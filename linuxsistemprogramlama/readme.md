# LINUX SISTEM PROGRAMLAMA

## Sistem Programlamaya Giriş

Ken Thompson işletim sistemini tasarlarken pek çok prensibi MULTICS'den almıştı. 

İlk defa MULTICS'te görülen başlıca özellikler aşağıdaki gibi sıralanabilir:

- Dosya Sistemi: Dosya ve dizinlerden oluşan ağaç yapısındaki hiyerarşi geliştirilmiştir. Sembolik link desteği gibi özelliklere de yer verilmiştir.
- Per-Process Stack: Çekirdek seviyesinde her process için ayrı bir stack alanı kullanılmıştır.
- Komut İşleyici - Kabuk: İlk defa komut işleyici (command processor) kullanıcı kipinde çalışan bir uygulama ile yapılmıştır. Bu fikir daha sonra kabuk şekliyle diğer işletim sistemlerinde ortaya çıkmıştır.
- Process Memory: Dosya işlemleri için IO katmanı yerine, dosyaların process'in adres alanı içerisine haritalanması ve dosya işlemlerinin standart bellek operasyonlarıyla yapılması, alt katmandaki işlemlerin ise işletim sistemi çekirdeği tarafından yerine getirilmesi sağlanmıştır. Günümüzdeki POSIX standartlarıyla konuşacak olursak, sistemdeki tüm dosyaların Memory-Mapped IO mmap ile kullanıldığındakine benzer bir uygulama söz konusudur.
- Dinamik Linkleme: Bazı yönleriyle günümüzdeki kullanımdan bile ileride olduğu söylenebilecek dinamik link desteği geliştirilmiştir.
- Çoklu İşlemci: Birden fazla CPU kullanımı desteklenmektedir.
- Hot-Swap: Çalışma zamanında sistemi kapatma ihtiyacı olmaksızın bellek, disk ve CPU eklenip çıkarılabilmektedir.



## Sistem Çağrıları

Kullanıcı kipinde çalışan bir uygulamanın, sistem çağrıları aracılığıyla işletim sistemi çekirdeğinden ihtiyaç duyduğu servisleri alabilmesi sağlanır.

Linux çekirdeğinde X86 mimarisi için yaklaşık 380 civarında sistem çağrısı bulunur.

### Sistem Çağrısı Nasıl Gerçekleşir?

<sub> Konunun devamında aksi belirtilmedikçe verilen örnekler 32 bit Intel mimarisi için geçerlidir. </sub>

Kullanıcı kipindeyken herhangi bir sistem çağrısı yapıldığında INT `0x80` makine dili kodu ile trap oluşturulur.

Aynı zamanda talep edilen sistem çağrısının numarası, `EAX` yazmacına yazılır.

32 bitlik Intel platformu için Linux çekirdek versiyonu 2.3.31 ve sonrası, maksimum 6 sistem çağrısı parametresini desteklemektedir. Bu parametreler sırasıyla `EBX`, `ECX`, `EDX`, `ESI`, `EDI` ve `EBP` yazmaçlarında saklanır.


### Mimari Bağımlılığı

Sistem çağrılarının doğrudan işlemci mimarisine bağımlıdir.

Örnek olarak Intel 32 bitlik işlemcilerde INT `0x80` ile trap oluşturulurken, ARM mimarisinde aynı işlem supervisor call `SVC` ile yapılır.

Benzer şekilde Intel mimarisinde yapılan sistem çağrısının numarası için `EAX` yazmacı kullanılırken, ARM mimarisinde `R8` yazmacı kullanılır. ARM mimarisinde sistem çağrısına ait 4 adede kadar parametre, `R9`, `R10`, `R11` ve `R12` yazmaçlarına aktarılır. 4 adetten fazla parametre geçilmesi gerektiğinde, bellek üzerinde veri yapısı hazırlanarak bu bölümün adresi geçirilir.


### Sistem Çağrısı Nasıl Yapılır?

Sistem çağrıları, `glibc` kütüphanesindeki *wrapper* fonksiyonlar üzerinden kullanılır.

`glibc` kütüphanesi, üzerinde çalıştığı çekirdek versiyonuna göre, hangi Linux sistem çağrısını yapacağını belirler.

> Bazı durumlarda ise bundan daha fazlasını yaparak, üzerinde çalışılan çekirdek versiyonunda hiç desteklenmeyen bir özelliği de sunuyor olabilir. Örnek olarak, Linux 2.6 versiyonuyla birlikte gelen POSIX Timer API'nin olmadığı Linux 2.4 versiyonu üzerinde çalışan ve aynı anda pek çok timer kullanan bir uygulamanız var ise, glibc çekirdek tarafından alamadığı desteği kullanıcı kipinde her timer için bir thread açarak sağlar. Elbette timer sayınız fazla ise bu çok yavaş bir çözüm olur ancak uygulamanın çalışmasını da mümkün kılar.


### Sistem Çağrıları → Glibc Fonksiyonları İlişkisi

Pek çok sistem çağrısı, aynı isimdeki `glibc` *wrapper* fonksiyonları üzerinden çağrılmaktadır.

Örnek olarak sistem çağrılarını takip etmede sık kullanacağımız strace aracı ile basit bir kodun sistem cagrilarini takip edelim:

```c
  printf("sa");
```

main icinde yazilmis ve gerekli kutuphaneler eklenmis sekilde bu kodu `strace ./deneme` olarak calistirdigimizda cikan ciktidaki karsilik soyledir:

```
[beyza@fedora oba]$ strace ./deneme
[REDACTED]
write(1, "sa", 2sa)                       = 2
[REDACTED]
+++ exited with 0 +++
```
Burada kastedilen glibc içerisindeki `write()` fonksiyonu değil, `write` sistem çağrısıdır.

Strace üzerinden sistem çağrısına geçirilen argümanları ve geri dönüş değerini (2) görmekteyiz.

> Normalde bu yöntemle sistem çağrılarının ismi değil numarası izlenebilir. Strace uygulaması elde ettiği sistem çağrısı numarasını kendi veritabanında arayıp bizler için daha okunabilir bir formda gösterir.


### Performans

Sistem çağrıları normal fonksiyon çağrılarına oranla oldukça yüksek maliyetli işlemlerdir.

Her sistem çağrısında uygulamanın o anki durumunun saklanması, çekirdeğin işlemcinin kontrolünü ele alması ve ilgili sistem çağrısı ile çekirdek kipinde talep edilen işlemleri gerçekleştirmesi, sonra ilgili uygulamanın tekrar çalışma sırası geldiğinde, uygulamanın saklanan durumunun yeniden üretilip işlemlerin kaldığı yerden devamının sağlanması gereklidir.


Özellikle x86 tabanlı mimarilerde `SYSENTER` özel yolu sayesinde sistem çağrılarının hızlanmasını sağladık. Ancak bu yeterli olacak mıdır?

Bir çok uygulamada, özellikle `gettimeofday()` gibi sistem çağrılarının çok sık kullanıldığını görürürüz.

Uygulamalarınızı `strace` ile incelediğinizde, bilginiz dahilinde olmayan pek çok farklı `gettimeofday()` çağrısını yapıldığını görebilirsiniz.

`glibc` kütüphanesinden kullandığınız bazı fonksiyonlar, internal olarak bu fonksiyonaliteyi kullanıyor olabilir.

Java Virtual Machine gibi bir VM üzerinde çalışan uygulamalar için de benzer bir durum söz konusudur.

Görece basit bir işlem olmasına rağmen sık kullanılan bu operasyon yüzünden sistemlerde nasıl bir sistem çağrısı yükü oluşmaktadır? sorusunu kendimize sorabiliriz.


### Linux Dynamic Linker/Loader: ld.so

Paylaşımlı kütüphaneler kullanan uygulamaların çalıştırılması sırasında, Linux Loader tarafından gereken kütüphaneler yüklenerek uygulamanın çalışacağı ortam hazırlanır. En basit Hello World uygulamamız bile `libc` kütüphanesine bağımlı olacaktır.

`ldd` ile uygulamanın linklenmiş olduğu kütüphanelerin listesini alabiliriz:

```
[beyza@fedora oba]$ ldd deneme
	linux-vdso.so.1 (0x00007fffd0ba0000)
	libc.so.6 => /lib64/libc.so.6 (0x00007f0c6f800000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f0c6fa7a000)
```

`Görüldüğü üzere `libc` ve `ld.so` bağımlılıkları listelendi. Fakat `linux-vdso.so.1` kütüphanesi nedir?`

### Virtual DSO: linux-vdso.so.1

linux-vdso.so.1 sanal bir Dnamically Linked Shared Object dosyasıdır. Gerçekte böyle bir kütüphane dosya sistemi üzerinde yer almaz. Linux çekirdeği, çok sık kullanılan bazı sistem çağrılarını, bu şekilde bir hile kullanarak kullanıcı kipinde daha hızlı gerçekleştirmektedir.

Örnek olarak, sistem saati her değişiminde sonucu tüm çalışan uygulamaların adres haritalarına da eklenmiş olan özel bir bellek alanına koyarsa, `gettimeofday()` işlemi gerçekte bir sistem çağrısına yol açmadan kullanıcı kipinde tamamlanabilir.



## API ve ABI

Yazılım geliştiriciler açısından uygulamaların tüm Linux dağıtımlarında ve desteklenen tüm mimarilerde çalışması temel hedeflerden biridir. İdeal durumda, geliştirilen yazılımın belirli bir dağıtım veya mimariye bağımlı olmayıp, taşınabilir olması beklenir.

Sistem seviyesinden bakıldığında taşınabilirlikle ilgili 2 farklı tanım bulunur. Bunlardan ilki *Application Programming Interface* (API) diğeri ise *Application Binary Interface* (ABI) şeklindedir.

### API

API, uygulamaların kaynak kod seviyesinde birbirleriyle iletişim kurabilmelerine imkan sağlayan, önceden anlaşılmış arayüzler olarak tanımlanabilir.

Genellikle her bir API, daha karmaşık ve alt seviye detaylar içeren bir sürecin, çeşitli arayüzlerle (fonksiyon çağrıları vb.) soyutlanması sağlar.

Bu şekildeki bir soyutlama üzerinden kullanılan API'yi sağlayan yazılım bileşenleri güncellense ve alt tarafta yapılan işlemlerle ilgili yöntemler değiştirilse dahi, API seviyesinde aynı arayüz sağlandığı müddetçe diğer uygulamalar için kaynak kod seviyesinde bir değişiklik yapılmasına gerek olmayacaktır.


### ABI

API kaynak kod seviyesinde bir arayüz tanımlarken ABI yazılım bileşenleri arasında belirli bir mimari için obje kod (binary) arayüzlerini tanımlamaktadır.

ABI uygulama bileşenleri arasında binary compatibility'yi sağlar. Bu uyum korunduğu müddetçe aralarında etkileşim bulunan yazılım bileşenlerinin versiyonları değişse de yeniden derlenmeye ihtiyaç duymaksızın eskisi gibi çalışmaya devam ederler.

Fonksiyonların nasıl çağrılacağı, parametrelerin nasıl geçirileceği, yazmaçların kullanım şekilleri, sistem çağrılarının gerçekleştirilme biçimi, uygulamların linklenmesi, binary obje dosya formatı gibi konular ABI içerisinde incelenir. Tüm mimariler için ortak bir ABI oluşturmak mümkün değildir (Java Virtual Machine gibi çözümler ayrı bir başlıkta değerlendirilebilir).

Yazılım geliştirme sürecinde ABI varlığını pek hissettirmez. Kullanılan toolchain içerisindeki araçlar (derleyici, linker vb.) hedef platform için uygun ABI ile kod üretirler.



## Dosya Sistemi

Genel olarak UNIX tabanlı sistemler için şöyle bir özdeyiş vardır: *UNIX sistemlerinde her şey birer dosyadır; eğer bir şey dosya değilse o zaman process'dir.*

Bu yaklaşım Linux için de geçerlidir. Bazı dosyaların diğerlerinden daha özel anlamlarının da olabilmesiyle birlikte (aygıt dosyaları, domain soketler, pipe, link vb.), pek çok işlem dosya sistematiği içerisinde kalınarak gerçekleşmektedir.

















