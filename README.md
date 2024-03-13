# str_id_consteval

This is a header-only C++ 20 consteval version of the MurmurHash32 algorithm just for strings (const char*) providing the following function:

``` C++
consteval uint32_t strID(const char* str);
```

and ensuring the conversion from str to unsigned is performed during the **compilation**, not at runtime.

The code has been adapted from the original code 32bits version at https://github.com/aappleby/smhasher, I just made a self-contained consteval version.

I normally use strings for identifications, but when comparing, searching or using as keys of containers, I convert them to uint32_t and the mapping is done using the MurmurHash32 algorithm. Remember the mapping from const char* to uint32_t is **not** unique and there is a non-zero possibility to have collisions.

``` C++

// Helper class to store the id associated to a string.
struct StrId {
	uint32_t id = 0;
	StrId( ) = default;
	constexpr StrId( uint32_t new_id ) : id( new_id ) { };
	StrId(const char* text) : id(getID(text)) {}

	operator uint32_t() const { return id; }
	bool operator==(const StrId& other) const { return id == other.id; }
	bool operator!=(const StrId& other) const { return !(id == other.id); }
	bool operator==(uint32_t other) const { return id == other; }
	bool operator!=(uint32_t other) const { return !(id == other); }
	const char* c_str() const { return getStr( id ); }
};

// --------------------------------------------------
unsigned int getID(const char* msg) {
  size_t len = strlen(msg);
  if (len == 0) return 0;
  return getID(msg, len);
}

unsigned int getID(const void* data, size_t nbytes) {
  unsigned int id;
  MurmurHash3_x86_32(data, static_cast<int>(nbytes), 0, &id);
  return id;
}

...
StrId user_name( get_name_from_somewhere() );
if( user_name == strId("admin")) { 
  printf( "it's the admin %08x\n", user_name.id);
}

```

This becomes, in **debug** builds

```
  if (user_name == strID("admin")) {
00007FF6F11161E8  mov         edx,5525E156h  				// <-- Becomes an inmediate
00007FF6F11161ED  lea         rcx,[user_name]  
00007FF6F11161F1  call        StrId::operator== (07FF6F1079DBCh)  
00007FF6F11161F6  movzx       eax,al  
00007FF6F11161F9  test        eax,eax  
00007FF6F11161FB  je          __$EncStackInitStart+0BAh (07FF6F111620Ch)  
	  dbg( "it's the admin %08x\n", user_name.id);
00007FF6F11161FD  mov         edx,dword ptr [user_name]  
00007FF6F1116200  lea         rcx,[string "it's the admin %08x\n" (07FF6F11E9E68h)]  
00007FF6F1116207  call        dbg (07FF6F1079B05h)  
  }
```

and in release, the operator == becomes inline.

```
  if (user_name == strID("admin")) {
00007FF69A4922CE  cmp         dword ptr [rsp+30h],5525E156h  // <-- Becomes a cmp against an inmediate cte
00007FF69A4922D6  jne         wWinMain+0C9h (07FF69A4922E9h)  
	  printf( "it's the admin %08x\n", user_name.id);
00007FF69A4922D8  mov         edx,5525E156h
00007FF69A4922DD  lea         rcx,[string "it's the admin %08x\n" (07FF69A4C3438h)]  
00007FF69A4922E4  call        printf  
  }
```

# Install

No dependencies required, just copy the .h to your project.
