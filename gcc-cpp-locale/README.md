## GCC C++ locale support

Consider the following simple C++ program:

```cpp
#include <codecvt>
#include <iostream>
#include <locale>

int
main()
{
    auto const &locale_name = "en_US.UTF-8";
    auto utf8_locale = std::locale {
        std::locale{locale_name},
        new std::codecvt_utf8<wchar_t>
    };
    try {
        std::locale::global(utf8_locale);
    } catch (std::runtime_error &) {
    }
}
```

On all OS I can test this works fine, except illumos:

```
$ ./test
terminate called after throwing an instance of 'std::runtime_error'
  what():  locale::facet::_S_create_c_locale name not valid
Abort (core dumped)
```

The problem is that GCC on illumos for C++ uses the "generic" locale backend,
which only supports the `C` locale, and aborts when using anything else:

<https://github.com/gcc-mirror/gcc/blob/releases/gcc-13.2.0/libstdc%2B%2B-v3/config/locale/generic/c_locale.cc#L241-L248>

The original thread which discovered this issue in mkvmerge is here:

<https://smartos.topicbox.com/groups/smartos-discuss/T16289ae73cd17a35/stock-mkvtoolnix-crashes-on-2023-4-0-with-locale-error>

## Fix

I've ported the DragonFly BSD locale backend to illumos, and the pkgsrc GCC
13.2.0 now has these fixes applied:

<https://github.com/TritonDataCenter/pkgsrc-extra/commit/51e7c8f7e54789951d1f95c141eeeaf9502c66ba>
<https://github.com/TritonDataCenter/pkgsrc-extra/commit/bb76430cada3c07a2f5b13938761003b73425980>

Due to missing support for `localeconv_l()` there is currently no support for
`LC_MONETARY` and `LC_NUMERIC`, but otherwise this seems to work well.

```
$ ./test
$
```

and the original reported issue is also resolved in mkvmerge:

```
$ env LANG=es_ES.UTF-8 mkvmerge
mkvmerge v82.0 ('I'm The President') 64-bit
Error: no fue asignado ningún nombre al archivo de salida.

mkvmerge -o salida [opciones generales] [opciones1] <archivo1> [@opciones-archivo.json] ...
...
Por favor, lea la página principal o la documentación HTML para
mkvmerge. En ella, se explican ciertos aspectos de una manera
más detallada que no son tan obvias desde este listado.
```

During testing there was no observed regressions in a full bulk build, and
these patches have been used in the pkgsrc trunk builds for a couple of months
now (as of May 2024) with no reported issues.
