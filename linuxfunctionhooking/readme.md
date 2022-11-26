# LINUX FUNCTION HOOKING

## What Are Shared Libraries?

Paylaşılan kitaplıklar, yürütülebilir bir dosya üretmenin son adımlarında bağlanan önceden derlenmiş C kodudur.  Daha sonra kendi kodunuzu yazarken kullanılabilecek işlevler, rutinler, sınıflar, veri yapıları vb. gibi yeniden kullanılabilir özellikler sağlarlar.


Linux'un İçerdiği Ortak Paylaşılan Kitaplıklar şunlardır:

- libc : The standard C library.
- glibc : GNU Implementation of standard libc.
- libcurl : Multiprotocol file transfer library.
- libcrypt : C Library to facilitate encryption, hashing, encoding etc.

Paylaşılan kitaplıklar hakkında bilinmesi gereken en önemli şey, programların çalışma süresi boyunca ihtiyaç duyduğu çeşitli işlevlerin adreslerini içermeleridir.


Örneğin, dinamik olarak bağlı bir yürütülebilir dosya bir read() sistem çağrısı verdiğinde, sistem libc paylaşımlı kitaplığından read() adresini arar. Artık libc'nin read() için işlev parametrelerinin sayısını ve türünü belirten ve karşılığında belirli bir veri türü bekleyen iyi tanımlanmış bir tanımı vardır. Genellikle sistem bu işlevleri nerede arayacağını bilir, ancak sistemin bu işlevleri nerede arayacağını ve bunları kötü amaçlar için nasıl kullanabileceğimizi kontrol edebiliriz.


TL;DR: Paylaşılan Kitaplıklar, daha sonra belirli işlevleri gerçekleştirmek için çağrılabilen işlev tanımlarını içeren derlenmiş C kodudur.  Dinamik olarak bağlantılı yürütülebilir dosyaları çalıştırdığımızda, sistem bu kitaplıklardaki ortak işlevlerin tanımlarını arar.

> Linuxta dinamik baglayici/yukleyicinin adi nedir? *ld.so, ld-linux.so*



## Getting A Tad Bit Technical 

Oncelikle, ls komutu icin gerekli olan dinamik olarak linklenmis kutuphanelere bakalim. Bunun icin terminale su yazilabilir:

```
ldd `which ls`
```

ya da fish shell kullaniliyorsa:

``` 
ldd (which ls)
```

cikan sonuc:

```
# ldd /bin/ls        
         linux-gate.so.1 (0xb7f54000)        
         libselinux.so.1 => /lib/i386-linux-gnu/libselinux.so.1 (0xb7ed7000)        
         libc.so.6 => /lib/i386-linux-gnu/libc.so.6 (0xb7cf9000)         
         libdl.so.2 => /lib/i386-linux-gnu/libdl.so.2 (0xb7cf3000)
         libpcre.so.3 => /lib/i386-linux-gnu/libpcre.so.3 (0xb7c7a000)
         /lib/ld-linux.so.2 (0xb7f56000)
         libpthread.so.0 => /lib/i386-linux-gnu/libpthread.so.0 (0xb7c59000)

```

<sub> *bu cikti x86 kali'den alinmistir. 64 bitlik bir sistemde farkli kutuphaneler olabilir.* </sub>

Bu ciktida `libc.so.6` isimli ve `/lib/i386-linux-gnu/libc.so.6` konumunda bir kutuphane buluyoruz.
*Not: Bunlar, sistemde başka bir yerde bulunan gerçek paylaşılan kitaplık dosyalarına yalnızca sembolik bağlantılardır.*


Man page'de sunlari bulabiliriz:

- *Using the directories specified in the DT_RPATH dynamic section attribute of the binary if  present  and  DT_RUNPATH  attribute does not exist.  Use of DT_RPATH is deprecated.*
- *Using the environment variable LD_LIBRARY_PATH, unless the executable is being run in secure-execution mode (see  below),  in which case this variable is ignored.*
- *Using  the directories specified in the DT_RUNPATH dynamic section attribute of the binary if present.  Such directories  are searched  only to find those objects required by DT_NEEDED (direct dependencies) entries and do not apply to  those  objects' children,  which  must themselves have their own DT_RUNPATH entries.  This is unlike DT_RPATH, which is applied  to  searches  for all children in the dependency tree.*
- *From the cache file /etc/ld.so.cache, which contains a compiled list of candidate shared objects previously found in  the  augmented  library  path.  If, however, the binary was linked with the -z nodeflib linker option, shared objects in the default paths are skipped.  Shared objects installed in hardware capability directories (see below) are preferred to other  shared objects.*
- *In  the  default path /lib, and then /usr/lib.  (On some 64-bit   architectures, the default paths for 64-bit shared objects  are /lib64,  and  then  /usr/lib64.)  If the binary was linked with the -z nodeflib linker option, this step is skipped.*

Bu yazilar arasinda onemli olan uc nokta var. Bunlar:

- The LD_PRELOAD environment variable.
- The --preload command-line option when invoking the dynamic linker directly.
- The /etc/ld.so.preload file.

Diger paylasilan kitapliklardan **ONCE** yuklenen kendi paylasilan nesnelerimizi belirtmemize izin verdigi icin 1 ve 3 noktalariyla daha cok ilgileniyoruz ve benzer PATH saldirilarina cok benzer sekilde bunlari acik olarak kullanip somurecegiz.

>  What environment variable let's you load your own shared library before all others? *LD_PRELOAD*
>  Which file contains a whitespace-seperated list of ELF shared objects to be loaded before running a program? */etc/ld.so.preload*
> If both the environment variable and the file are employed, the libraries specified by which would be loaded first? *environment variable*



## Putting On Our Coding Hats

Cok basit bir program ile baslayalim.

```c
#include <unistd.h>
int main()
{
  char str[12];
  int s;
  s=read(0, str,13);      
  write(1, str, s);                          
  return 0;
}
```

Derleyip calistirdigimizda, okudugu yaziyi cikti olarak bize sundugunu goruruz. Gayet basit bir sekilde `stdin`'den veri alir ve `stdout`tan yazdirir. Simdi sahnenin arkasinda olana bakalim:

Normal şartlar altında, dinamik bağlayıcı karşılaştığında `write()` işlevi, adresini standart paylaşılan kitaplıklarda arar. İlk oluşumla karşılaştığında `write()`, bağımsız değişkenleri işleve iletir ve buna göre bir çıktı döndürür. 

Simdi bunu somurme vakti.

man page'den `write()` fonksiyonu tanimini ssize_t `write(int fd, const void *buf, size_t count);` olarak `ssize_t` return tipi ile aliyoruz. 

Zararli fonksiyonumuzun da hook'lamaya calistigimiz orijinal fonksiyonla ayni fonksiyon tanimina ve donus tipine sahip olmasi cok onemlidir. Simdi kendi kotu amacli paylasilan kitapligimizi yazmaya baslayabiliriz:

```c
#include <stdio.h>
#include <unistd.h>
#include <dlfcn.h>
#include <string.h>
ssize_t write(int fildes, const void *buf, size_t nbytes)
{
     ssize_t (*new_write)(int fildes, const void *buf, size_t nbytes); 
     ssize_t result;
     new_write = dlsym(RTLD_NEXT, "write");
     if (strncmp(buf, "Hello World",strlen("Hello World")) == 0)
     {
          result = new_write(fildes, "Hacked 1337", strlen("Hacked 1337"));
     }
     else
     {
          result = new_write(fildes, buf, nbytes);
     }
     return result;
}
```

Simdi sira kodun aciklamasinda:

- Ilk olarak gerekli header files'lari ekliyoruz. 
- Daha sonra hooklamaya calistigimiz fonksiyonla tamamen ayni fonksiyon tanimina ve donus turune sahip bir fonksiyon olusturmamiz gerekiyor. bunun nedeni, fonksiyonu cagiran programlarin br dizi parametre gondermesi ve karsiliginda belirli bir cikti turu beklemesi ve hatalara neden olmasidir.
- Daha sonra cok onemli bir sey yapiyoruz. `new_write` isimli, daha sonra write adresinin yerine gececek `write()` fonksiyonunun yerine bir fonksiyon olusturuyoruz.
- Ayrica donus degerini toplamak icin `result` isimli bir value olusturuyoruz. 
- Son olarak, orijinal `write()` islevinin konumunu daha once olusturdugumuz islev isaretcisine kaydediyoruz. Standart paylasilan kitapliklardan (RTLD_NEXT olarak belirtildigi gibi) bir sonraki yazma isleminin adresini almak icin dlsym islevini kullaniriz. 


Suana kadarki adimlar standartti. Asagida belirttiklerim farkli hooklar icin de neler olabilecegidir.:

- *Now we have some fun. Here, we compare the string buffer passed to the function to see if it equals "Hello World". If it does, we call  the original write() function using the function pointer but replace it with our own string and store the result returned. You can do anything you feel like : generate logs, trigger other conditions, create connections if certain conditions are met and so on. Feel free to play around this part!*
- *If the conditions aren't met, however, we simply pass all the parameters to the original function via our function pointer, and store the result.(
- *We finally return the result to the calling function.(


## Let's Gooooooooooooooo

Simdi sirada kitapligi derleyip calistirip neler oldugunu gorme vakti.

### Compiling Our Program

`gcc -ldl malicious.c -fPIC -shared -D_GNU_SOURCE -o malicious.so `

*Not: eger symbol lookup error alirsan sunu dene:*

`gcc malicious.c -fPIC -shared -D_GNU_SOURCE -o malicious.so -ldl`

Simdi kodu anlayalim:

- gcc: gcc
- -ldl: dinamik baglanti kitapligi olarak da bilinen libdl'ye karsi baglanti
- malicious.c: malicious.c
- -fPIC: konumdan bagimsiz kod olustur (daha genis aciklama [burada](https://stackoverflow.com/questions/966960/what-does-fpic-mean-when-building-a-shared-library))
- -shared: compiler'a yürütülebilir bir dosya oluşturmak için diğer nesnelerle bağlantılı olabilecek Shared Object olusturmasini soyler.
- -D_GNU_SOURCE:  RTLD_NEXT sıralamasını kullanmamıza izin veren #ifdef koşullarını karşılamak için belirtilmiştir. İsteğe bağlı olarak bu flag #define _GNU_SOURCE eklenerek değiştirilebilir.
- -o: -o
- -malicious.so: malicious.so

Simdi malicious.so dosyasini olusturdugumuza gore sirada ne var ona bakalim.

### Pre-Loading Our Shared Object

Artık Paylaşılan Nesne dosyamız hazır olduğuna göre, işlevimizi başarılı bir şekilde bağlamak için onu diğer paylaşılan kitaplık nesnelerinden önce önceden yüklememiz gerekiyor. Bunu yapmak için iki yöntemimiz var:

- `LD_PRELOAD` kullanmak:

`export LD_PRELOAD=$(pwd)/malicious.so`

- `/etc/ld.so.preload` dosyasini kullanmak:

`sudo sh -c "echo $(pwd)/malicious.so > /etc/ld.so.preload"`

su bicim olmus mu kontrol edebilirsin:

```
$ ldd /bin/ls
     linux-gate.so.1 (0xb7fc0000)
     /home/whokilleddb/malicious.so  (0xb7f8f000)
     libselinux.so.1 => /lib/i386-linux-gnu/libselinux.so.1 (0xb7f43000)
     libc.so.6 => /lib/i386-linux-gnu/libc.so.6 (0xb7d65000)
     lbdl.so.2 => /lib/i386-linux-gnu/libdl.so.2 (0xb7d5f000)
     libpcre.so.3 => /lib/i386-linux-gnu/libpcre.so.3 (0xb7ce6000)
     /lib/ld-linux.so.2 (0xb7fc2000)
     libpthread.so.0 => /lib/i386-linux-gnu/libpthread.so.0 (0xb7cc5000)
```

Şimdi senaryo şuna benzer: Program, tüm parametreler yerindeyken write() işlevine bir çağrı yapar. Bununla birlikte, `write()` öğesinin libc tanımına gitmek yerine, dinamik bağlayıcı `write()` öğesinin İLK OLUŞUMUNU bulduğundan ve işini yapmasına izin verdiğinden, bizim durumumuzda basit bir karşılaştırma işlemi olan, kötü amaçlı paylaşılan nesnemize gider. true ise, kötü amaçlı/kurcalanmış çıktıyı döndürür, aksi takdirde parametreleri libc içindeki gerçek işleve aktarır ve elde edilen çıktıyı programa geri iletir.

PATH Hijacking yontemine fazlasiyla benziyor.

Su dakikadan sonra ne zaman `write()` cagirmak istersek (ornegin pythondaki `print()` ya da c'deki `printf()` yazmak istedigimiz sey yerine daha onceden belirledigimiz `Hacked 1337` yazisini goruruz. 

