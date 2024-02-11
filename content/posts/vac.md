+++
title = "Reversing VAC: Initalization (1)"
date = "2023-01-04"
author = "absceptual"
description = "An introduction to the dissection of VAC."
+++

As a disclaimer, the code presented in this writeup is moderately or heavily beautifed for the reader's discretion.
Some topics may be omitted for the sake of clarity, or because they lack revelance to the topic at hand.
Some information in this writeup may be inaccurate due to my inexperience with reversing anticheats. If you believe there is something that should be changed, feel free to let me know.
Lastly, this is not an attack on Valve. This research is simply for educational purposes, and not intended for malicious usage.

## What is VAC?
VAC (also known as Valve Anti-Cheat) is a security measure deployed by Valve to deter and stop potential cheating attempts and ensure the integrity of VAC-protected games.
VAC should be thought, not as a single binary that contains all the code needed to prevent cheaters, but instead a series of modules injected into the target process that work together. These modules never hit your disk, but instead are dynamically loaded at the server's disposal.

In this writeup, we'll be doing a brief overview of the code used to load and setup these modules, and a POC that demonstrates how we can obtain these modules.

``CClientModuleManager::LoadModule``
This function is responsible for taking a CRC32 value for identifying the target module to be loaded, flags that dictate how the module should be loaded and a few other arguments used by the function.

{{< code language="CPP" title="CClientModuleManager::LoadModule" expand="Show" collapse="Hide" isCollapsed="false" >}}
unsigned int __thiscall CClientModuleManager::LoadModule(int crc, uint8_t flags, int a4, int a5, int scan_id, int a7, int a8, int a9, size_t size, void* buffer)
{
    
    // Retrieves an index into a module array from the CRC provided in our function arguments
    auto result = 0;
    auto index = this->GetModuleFromCRC( &crc );
    auto module = this->module_table->modules[ index ];
    
    if ( !index || module )
    {
        *( int* )( buffer ) = 0;
        result = 2;
    }
    else
    {
        // Allocates some memory and calls valve::get_entry_point to map the module
        auto v14 = *buffer;
        if ( (unsigned int)*a11 < 0x58 )
            v14 = 88;
        
        auto _buffer = v14;
        auto allocator = CStdMemAlloc( );
        *size = allocator->NewVariable( _buffer );
        if ( valve::get_entry_point( module, flags )
        {
            if ( *size )
            {
                // Executes the entry point (runfunc@20) of the module and stores the result
                sub_63C0CB50((__m128i *)*size, 0, _buffer);
                sub_63C0CB50((__m128i *)v20, 0, 0x50u);
                ms_exc.registration.TryLevel = 0;
                module->m_nLastResult = ( entry_t )module->m_pEntryPoint)(scan_id, arg2, arg3, *size, &_buffer )
                ms_exc.registration.TryLevel = -1;
            }
            else
            {
                *_buffer = 0;
                module->m_nLastResult = ALLOCATION_FAILED;
            }
        }
        else
            *_buffer = 0;
        
        result = module->m_nLastResult;
        if ( result == 1 && _buffer > *buffer )
        {
            result = module->m_nLastResult = 15;
            _buffer = 0;
        }
        if ( result != 16 )
        {
            // More information propagating and error checking
            module->m_pModule = static_cast< CMappedModule* >( 0x63F68064 ); // Points to a CMappedModule structure when manual mapped
            module->m_hModule = static_cast< HMODULE >( 0x63F68060 ); // Points to the module base if valve::get_entry_point is called with the LoadLibrary flag
            if ( !result
               && !ASSERT(
              (int)"c:\\buildslave\\steam_rel_client_hotfix_win32\\build\\src\\SteamServiceClient\\servicemodulemanagerbase.cpp",
              251,
              "pModule->m_nLastResult != k_ECallResultNone") )
            {
                __debugbreak();
            }
            *buffer = _buffer;
            result = module->m_nLastResult;
        }
        if ( !result__
            && !ASSERT(
            (int)"c:\\buildslave\\steam_rel_client_hotfix_win32\\build\\src\\SteamServiceClient\\servicemodulemanagerbase.cpp",
            255,
            "pModule->m_nLastResult != k_ECallResultNone") )
        {
            __debugbreak();
        }
        v18 = module->m_nLastResult;
        if ( (flags & 4) != 0 )
          (*((void (__thiscall **)(CClientModuleManager *, int))CClientModuleManager->__vfptr + 1))(
            CClientModuleManager,
            crc);
        result = v18;
    }
    return result;
}
{{< /code >}}

After verifying that the requested module exists and some extra data is initalized ``valve::get_entry_point`` is called. This function takes the module data provided by ``CClientModuleManager::GetModuleFromCRC`` and the flags used to call our first function and gets the entry point of the requested module.

{{< code language="CPP" title="GetEntryPoint" expand="Show" collapse="Hide" isCollapsed="false" >}}
bool __stdcall GetEntryPoint(CValveModule *this, char injection_flags)
{
  CValveModule *_this; // esi
  internal_module *v4; // eax
  int v5; // eax
  int *v6; // eax
  CValveModule *v7; // edi
  CValveModule *v8; // ecx
  CValveModule *v9; // edx
  const CHAR *v10; // eax
  int v11; // eax
  int *v12; // eax
  bool v13; // zf
  int v14; // ecx
  int *v15; // eax
  char v16[40]; // [esp+8h] [ebp-28h] BYREF

  _this = this;
  if ( this->m_pEntryPoint )
    return 1;                                   // No need to map if the entry point has already been set
  if ( this->m_pRawModule && this->m_nSize )
  {
    if ( this->m_pModule
      && !ASSERT(
            (int)"c:\\buildslave\\steam_rel_client_hotfix_win32\\build\\src\\SteamServiceClient\\servicemodulemanagerbase.cpp",
            542,
            "pModule->m_pModule == NULL") )
    {
      __debugbreak();
    }
    if ( _this->m_hModule
      && !ASSERT(
            (int)"c:\\buildslave\\steam_rel_client_hotfix_win32\\build\\src\\SteamServiceClient\\servicemodulemanagerbase.cpp",
            543,
            "pModule->m_hModule == NULL") )     // Checks that the modules exist
    {
      __debugbreak();
    }
    if ( authenticate_module(_this->m_pRawModule, _this->m_nSize) ) // Does some file checking and some shit with public/private keys
    {
      valve::unload_module(_this);              // detected file patching or something of the sort
      _this->m_nLastResult = 11;
      return 0;
    }
    if ( (injection_flags & 2) != 0 ) // Manually map module
    {
      v4 = manual_map((IMAGE_DOS_HEADER *)_this->m_pRawModule, 0, 1);
      _this->m_pModule = (CMappedModule *)v4;
      if ( v4 )
      {
        v5 = get_export((int)v4, (int)"_runfunc@20");
        _this->m_pEntryPoint = (void *)v5; 
        if ( !v5 )
          _this->m_nLastResult = 25;
      }
      else
      {
        _this->m_nLastResult = 22;
      }
    }
    else
    {                                           // loadlibrary
      this = 0;
      sub_63BC6430((int)v16, 0, 0, 0);
      _this->m_nLastResult = 0;
      if ( sub_63BD3690((int *)&this) )
      {
        sub_63BC7D30((int)v16, (int)_this->m_pRawModule, _this->m_nSize, _this->m_nSize, 0);
        v7 = (CValveModule *)&pszSubKey;
        v8 = (CValveModule *)&pszSubKey;
        if ( this )
          v8 = this;
        if ( (unsigned __int8)sub_63BD47F0(v16, v8, 0) )
        {
          v9 = (CValveModule *)&pszSubKey;
          if ( this )
            v9 = this;
          sub_63BC60E0((int *)&_this[1].m_pEntryPoint, (const char *)v9);
          if ( this )
            v7 = this;
          v10 = (const CHAR *)sub_63BD49E0((const char *)v7, 0);
          _this->m_hModule = (HMODULE)v10;
          if ( v10 )
          {
            v11 = get_export_load_library(v10, "_runfunc@20");
            _this->m_pEntryPoint = (void *)v11;
            if ( !v11 )
              _this->m_nLastResult = 23;
          }
          else
          {
            _this->m_nLastResult = 22;
          }
        }
        else
        {
          _this->m_nLastResult = 21;
        }
        sub_63BC5310(v16);
        v12 = CStdMemAlloc_ctor();
        (*(void (__thiscall **)(int *, CValveModule *, _DWORD))(*v12 + 28))(v12, this, 0);
      }
      else
      {
        _this->m_nLastResult = 19;
        sub_63BC5310(v16);
        v6 = CStdMemAlloc_ctor();
        (*(void (__thiscall **)(int *, CValveModule *, _DWORD))(*v6 + 28))(v6, this, 0);
      }
    }
    if ( !_this->m_pEntryPoint )
    {
      valve::unload_module(_this);
      return 0;
    }
    v13 = _this->m_pRawModule == 0;
    v14 = dword_63F68064;
    _this[1].m_hModule = (HMODULE)dword_63F68060;
    _this[1].m_pModule = (CMappedModule *)v14;
    if ( !v13 )
    {
      v15 = CStdMemAlloc_ctor();
      (*(void (__thiscall **)(int *, void *, _DWORD))(*v15 + 28))(v15, _this->m_pRawModule, 0);
      _this->m_pRawModule = 0;
    }
    return 1;
  }
  this->m_nLastResult = 12;
  return 0;
}
{{< /code >}}

The entry point, known as ``runfunc`` is called and afterwards some standard error checking is deployed.
This entry point and the setup function following it will be covered in part 2 of **Reversing VAC: Initalization.**

## **Dumping VAC modules**
After analyzing the ``valve::get_entry_point`` function, we see a conditional that checks whether or not to use manual mapping, or LoadLibrary to map our DLL. By bytepatching this jump, we can force Steam to use LoadLibrary every single time.

This allows us to hook LoadLibrary and dump the DLL path to disk. Please note that this DLL will not have the imports resolved, so it is up to the reader to figure out how to resolve it's imports

## Conclusion
![Hooking LoadLibrary to dump VAC modules](/img/modules.png)
