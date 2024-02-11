+++
title = "Breaking MinHook"
date = "2023-01-04"
author = "LabGuy94"
description = "Looking at MinHook source and seeing how we can prevent hooks being placed."
+++

## What is MinHook?
[MinHook](https://github.com/TsudaKageyu/minhook) is a popular library written in C to hook functions.

## Why do we care about preventing the placement of hooks?
Usermode anticheats can utilize this library to place hook on internal functions to detect cheats trying to call them.
Also if you are an anticheat developer, you prevent cheaters from using MinHook.

## Analysis
Due to the fact that MinHook is open source, we can look at the source code and search for vulnerabilities that can be exploited to prevent hooks from being placed.
Lets start at the ``MH_EnableHook`` function.


{{< code language="C" title="MH_EnableHook" expand="Show" collapse="Hide" isCollapsed="false" >}}
MH_STATUS WINAPI MH_EnableHook(LPVOID pTarget)
{
    return EnableHook(pTarget, TRUE);
}
{{< /code >}}

Well that was anti climactic. This function is just a wrapper for the ``EnableHook`` function. Lets jump into this function

{{< code language="C" title="EnableHook" expand="Show" collapse="Hide" isCollapsed="false" >}}
static MH_STATUS EnableHook(LPVOID pTarget, BOOL enable)
{
    MH_STATUS status = MH_OK;

    EnterSpinLock();

    if (g_hHeap != NULL)
    {
        if (pTarget == MH_ALL_HOOKS)
        {
            status = EnableAllHooksLL(enable);
        }
        else
        {
            UINT pos = FindHookEntry(pTarget);
            if (pos != INVALID_HOOK_POS)
            {
                if (g_hooks.pItems[pos].isEnabled != enable)
                {
                    FROZEN_THREADS threads;
                    status = Freeze(&threads, pos, ACTION_ENABLE);
                    if (status == MH_OK)
                    {
                        status = EnableHookLL(pos, enable);

                        Unfreeze(&threads);
                    }
                }
                else
                {
                    status = enable ? MH_ERROR_ENABLED : MH_ERROR_DISABLED;
                }
            }
            else
            {
                status = MH_ERROR_NOT_CREATED;
            }
        }
    }
    else
    {
        status = MH_ERROR_NOT_INITIALIZED;
    }

    LeaveSpinLock();

    return status;
}
{{< /code >}}

We can see that the real point where the hook is enabled is in the EnableHookLL function. We can see that for the function to get called the ``Freeze`` function return ``MH_OK``. Lets take a look at the ``Freeze`` function.

{{< code language="C" title="Freeze" expand="Show" collapse="Hide" isCollapsed="false" >}}
static MH_STATUS Freeze(PFROZEN_THREADS pThreads, UINT pos, UINT action)
{
    MH_STATUS status = MH_OK;

    pThreads->pItems   = NULL;
    pThreads->capacity = 0;
    pThreads->size     = 0;
    if (!EnumerateThreads(pThreads))
    {
        status = MH_ERROR_MEMORY_ALLOC;
    }
    else if (pThreads->pItems != NULL)
    {
        UINT i;
        for (i = 0; i < pThreads->size; ++i)
        {
            HANDLE hThread = OpenThread(THREAD_ACCESS, FALSE, pThreads->pItems[i]);
            if (hThread != NULL)
            {
                SuspendThread(hThread);
                ProcessThreadIPs(hThread, pos, action);
                CloseHandle(hThread);
            }
        }
    }

    return status;
}
{{< /code >}}
Off the bat we can see a call to ``EnumerateThreads`` decides if the rest of the code is executed. Lets take a look at the ``EnumerateThreads`` function.

{{< code language="C" title="EnumerateThreads" expand="Show" collapse="Hide" isCollapsed="false" >}}
static BOOL EnumerateThreads(PFROZEN_THREADS pThreads)
{
    BOOL succeeded = FALSE;

    HANDLE hSnapshot = CreateToolhelp32Snapshot(TH32CS_SNAPTHREAD, 0);
    if (hSnapshot != INVALID_HANDLE_VALUE)
    {
        THREADENTRY32 te;
        te.dwSize = sizeof(THREADENTRY32);
        if (Thread32First(hSnapshot, &te))
        {
            succeeded = TRUE;
            do
            {
                if (te.dwSize >= (FIELD_OFFSET(THREADENTRY32, th32OwnerProcessID) + sizeof(DWORD))
                    && te.th32OwnerProcessID == GetCurrentProcessId()
                    && te.th32ThreadID != GetCurrentThreadId())
                {
                    if (pThreads->pItems == NULL)
                    {
                        pThreads->capacity = INITIAL_THREAD_CAPACITY;
                        pThreads->pItems
                            = (LPDWORD)HeapAlloc(g_hHeap, 0, pThreads->capacity * sizeof(DWORD));
                        if (pThreads->pItems == NULL)
                        {
                            succeeded = FALSE;
                            break;
                        }
                    }
                    else if (pThreads->size >= pThreads->capacity)
                    {
                        pThreads->capacity *= 2;
                        LPDWORD p = (LPDWORD)HeapReAlloc(
                            g_hHeap, 0, pThreads->pItems, pThreads->capacity * sizeof(DWORD));
                        if (p == NULL)
                        {
                            succeeded = FALSE;
                            break;
                        }

                        pThreads->pItems = p;
                    }
                    pThreads->pItems[pThreads->size++] = te.th32ThreadID;
                }

                te.dwSize = sizeof(THREADENTRY32);
            } while (Thread32Next(hSnapshot, &te));

            if (succeeded && GetLastError() != ERROR_NO_MORE_FILES)
                succeeded = FALSE;

            if (!succeeded && pThreads->pItems != NULL)
            {
                HeapFree(g_hHeap, 0, pThreads->pItems);
                pThreads->pItems = NULL;
            }
        }
        CloseHandle(hSnapshot);
    }

    return succeeded;
}
{{< /code >}}

## Theorizing exploitation of vulnerability
Immediately we can see that this function has the same "one function determines if the rest gets executed" behavior. This time it is calling the ``CreateToolhelp32Snapshot`` function from the windows library ``kernel32.dll``

This means if we hook ``CreateToolhelp32Snapshot`` and make it always return ``INVALID_HANDLE_VALUE``, we can cause a chain reaction of failures leading to the hook never being enabled!
## Execution of vulnerability
We can use a simple hooking library like [LightHook](https://github.com/SamuelTulach/LightHook) to place a hook on the ``CreateToolhelp32Snapshot`` function. We can then make it always return ``INVALID_HANDLE_VALUE``.
We utilize the library in a dll that we inject into the target process. Here is what something like that would look like

Payload (DLL)
{{< code language="CPP" title="dllmain.cpp" expand="Show" collapse="Hide" isCollapsed="false" >}}
#include "LightHook.h"
#include <iostream>

HANDLE CreateToolhelp32SnapshotHook(
    DWORD dwFlags,
    DWORD th32ProcessID
)
{
	std::cout << "CreateToolhelp32SnapshotHook" << std::endl;
    return INVALID_HANDLE_VALUE;
}

void main()
{
	HookInformation hookInfo = CreateHook((void*)GetProcAddress(GetModuleHandleA("kernel32.dll"), "CreateToolhelp32Snapshot"), (void*)&CreateToolhelp32SnapshotHook);

    int status = EnableHook(&hookInfo);
	std::cout << "Hook Status: " << status << std::endl;
}

BOOL APIENTRY DllMain(HMODULE hModule,
    DWORD  ul_reason_for_call,
    LPVOID lpReserved
)
{
    switch (ul_reason_for_call)
    {
    case DLL_PROCESS_ATTACH:
        CreateThread(nullptr, NULL, (LPTHREAD_START_ROUTINE)main, hModule, NULL, nullptr);
    case DLL_THREAD_ATTACH:
    case DLL_THREAD_DETACH:
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
}
{{< /code >}}

Reciever of payload (test program)
{{< code language="CPP" title="main.cpp" expand="Show" collapse="Hide" isCollapsed="false" >}}
#include <iostream>
#include "minhook/minhook.h"

void functiontohook()
{
	std::cout << "Not Hooked!" << std::endl;
}

void hook()
{
	std::cout << "Hooked ):" << std::endl;
}

int main()
{
	getchar();
	if (MH_Initialize() != MH_OK)
	{
		std::cout << "Failed to initialize MinHook!" << std::endl;
		return 1;
	}
	if (MH_CreateHook(&functiontohook, &hook, NULL) != MH_OK)
	{
		std::cout << "Failed to create hook!" << std::endl;
		return 1;
	}
	std::cout << "Enable Hook Status: " << MH_StatusToString(MH_EnableHook(MH_ALL_HOOKS)) << std::endl;
	functiontohook();
	system("pause");
}
{{< /code >}}

Injecting this DLL into a process that calls the ``MH_EnableHook`` function leads to this return value and failure to place the hook.

![Hooking LoadLibrary to dump VAC modules](/img/minhookbreaking.png)

## Downsides
If the developer is smart enough (not very smart) they can just verify the ``MH_EnableHook`` function returns ``MH_OK`` and if it doesn't they can just exit the program.
``CreateToolhelp32Snapshot`` is also a frequently used function leading to breaking functionality in other parts of the program.
