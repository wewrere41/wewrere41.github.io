---
title: "C# Defensive Copy’den Kaçınma Yolları Ve Struct Micro Optimizasyonları"
description: "Defensive Copy Nedir?"
date: "2022-02-09T10:59:46.995Z"
categories: []
published: true
canonical_link: https://medium.com/@EnescanBektas/c-defensive-copyden-ka%C3%A7%C4%B1nma-yollar%C4%B1-ve-struct-micro-optimizasyonlar%C4%B1-d40641b5e85a
redirect_from:
  - /c-defensive-copyden-kaçınma-yolları-ve-struct-micro-optimizasyonları-d40641b5e85a
---

### Defensive Copy Nedir?

Bir value type field’ının readonly olarak işaretlenmesi , variable’ın yaşam süresi boyunca aynı değeri tutması demektir(immutable). Derleyici bu alanın değerinin başka bir property veya metot tarafından değiştirilip değiştirilemeyeceğini bilemediği durumlarda mutasyonları önleyebilmek için bir kopyasını oluşturur ve buna defensive copy denir.

Örneğin burada struct’ın biri readonly , diğeri ise readonly olmadan declare edilmiştir. Peki 2 si üzerinden de aynı metodu çalıştırırsak nasıl bir sonuç alırız?

undefined

Readonly olarak işaretlenen bir value type datasının(struct’ın içeriği dahil) değiştirilemeyeceğini biliyoruz. O halde sonuç 0 ve 1.

### Peki compiler arka planda ne yaparak bu değişime engel oldu?

Readonly işaretlenmeyen struct’ı çağırdığımız zaman **“ldflda”(** ”_Push the address of field of object obj on the stack._”**)** CIL instruction’ı ile çağırılıyor. Yani stackdeki adresini yolluyor ve bunun üzerinden işlem yapılıyor.

Fakat readonly işaretlenen struct’ı çağırdığımızda ldflda kullanılmaz.   
**“ldfld”(**_”Push the value of field of object (or value type) obj, onto the stack_”**)** ile stackdeki adresini göndermek yerine değerininin bir kopyasını yollar ve bunun üzerinde işlem yapılır. Ayrıca bu instruction diğerine göre daha yavaştır.

Yani **\_readOnlyStruct**.IncreaseX(); dediğimizde aslında struct’ın yeni bir kopyasını(**defensive copy**) çağırmış ve onun üzerinden bu metodu çalıştırmış olduk. Bu sayede readonly işaretlediğimiz struct’da bir değişiklik yaşanmasının önüne geçilmiş oldu.

undefined

### İyi peki bunun bir zararı var mı ?

Bunun ne ölçüde performans farkı yaratacağını açıklarken LegacyJIT ve RyuJIT olarak ikiye ayırmamız gerekli. Çünkü RyuJIT ile beraber bu problem ciddi seviyede optimize edilmiş durumda.

Bunu gözlemleyebilmek için büyük bir struct kullanarak LegacyJIT ve RyuJIT ile test edelim.

undefined

### LegacyJIT

undefined

### RyuJIT

undefined

Görüldüğü üzere LegacyJIT'de neredeyse 5 kat fark varken RyuJIT ile bu fark yok denecek kadar azalmış. Tabi ki buradaki 5 katlık fark da sabit değil. Daha az iterasyon veya daha küçük bir struct ile denersek fark gittikçe azalacaktır.

Çoğu zaman bu kadar büyük bir performans farkı yaratacak şekilde kullanmıyoruz. Ama micro optimizasyon’da olsa bunun üstesinden nasıl geleceğimizi bilmekte yarar var.

### Problemin kökünde yatan ne?

Buradaki problemin başrollerinde; struct’ın içinde readonly işaretlenmeden değer tutan bir property ve readonly olarak işaretlenmiş bir instance variable struct var.   
 Readonly struct variable’ı üzerinden N property’sine erişirken kendisi readonly işaretlendiği için eriştiği property’nin de readonly olup olmadığını kontrol ediyor. Property readonly işaretlenmediğinden dolayıda bu property’ye erişilirken struct’ın bir datasının değiştirilip değiştirilemeyeceğinden emin olamıyor ve önlem olarak bir kopya yaratıyor.

Örneğin N bu şekilde kullanılsaydı struct’ın içerisindeki başka bir datayı değiştirebilirdi. Biz bu ayrımı farkedebiliyoruz ama compiler ayırt edemiyor.

undefined

**Yani kabaca readonly üzerinden readonly olmayan property veya method’lara erişirken bu problemi yaşıyoruz**

Not: Auto property ({get;set;}) veya get-only auto propertylerde({get;}) bu problem yaşanmaz çünkü compiler arka planda property’yi readonly olarak işaretler.

undefined

#### Tamam peki bununla sınırlı mı?

Öyleyse İçinde sadece boş bir method olan struct’ın method’una readonly field’dan erişirsek de defensive copy yaratır mı, yoksa yaratmaz mı?

undefined

Cevabımız evet , yaratır.   
O zaman tüm durumları ele alalım.

### **A) Readonly Instance field’den erişim durumlarında**

**1)Property erişimleri**

undefined

readonly instance struct üzerinden başka bir field’ı, sabit değeri veya başka bir property’yi tutan property’ye erişmek defensive copy yaratır.

**2)Method erişimleri**

undefined

Bir değer döndürsün veya döndürmesin readonly instance struct üzerinden bir metoda erişmek defensive copy yaratır.

### **B) Struct’ın içinden erişim durumlarında**

Struct’ın içindeki readonly property veya readonly methodlar’dan readonly olmayan property veya methodlara erişim durumunda defensive copy yaratılır. Ama burda **“ldfld”** yerine **“ldobj”** komutu ile kendisini(this) kopyalar.

undefined

EmptyMethod arka planda compiler tarafından böyle çevrilir;

undefined

**1)Readonly property’den readonly olmayan property veya metodlara erişim**

undefined

**2)Readonly Method’dan readonly olmayan property veya metodlara erişim**

undefined

### **C) Ref keyword’ünün kullanım durumlarında**

Aşağıdaki kullanımlarda da ldobj ile struct kopyalanmaktadır.

**1)ref readonly local variable üzerinden readonly olmayan property veya metodlara erişim**

undefined

**2)ref readonly property veya metod üzerinden readonly olmayan property veya metoda erişmek**

undefined

### **D) in keyword’ünün kullanım durumlarında**

in keyword’ü aslında ref keyword’ünün readonly ile sarmalanmış halidir.

undefined

Aşağıdaki kullanımlarda da ldobj ile struct kopyalanmaktadır.

**1)in keyword’ü ile geçen parametre üzerinden readonly olmayan property veya metodlara erişmek**

undefined

### Peki bunlardan nasıl kaçınabiliriz?

#### readonly struct

Öncelikli olarak tavsiye edilen yöntem C# 7.2 ile beraber gelen readonly struct deklarasyonunu kullanmak.

undefined

readonly struct deklarasyonu ile compiler struct’ın içerisini kontrol ederek hepsinin readonly olarak işaretlenmiş olduğundan emin olur. Bu sayede compiler defensive copy yaratacağı durumlarda struct’ın readonly işaretlendiği görüp kopyalamayı atlar. Aynı zamanda dizayn açısından da kendini açıkladığı için öncelikli olarak tavsiye edilen yöntem budur.

#### readonly member

Bir diğer seçeneğimizde C# 8.0 ile gelen readonly member kullanımı. Bu yolla da erişmek istediğimiz property/methodları readonly işaretleyerek defensive copy’lerden kaçınmamız mümkün.

undefined

### Sonuç

Compiler’ın defensive copy’lerden kaçınması ve mikro’da olsa daha optimize kod yazmak için readonly modifier’larını doğru kullanmak önemli. Bunlara performans açısından dikkat etmesek bile hem daha doğru dizayn kararları açısından hemde kısmen hatalı bir design’ın çözümü için arka planda compiler’ın neler yaptığınının farkında olmak için bu bilgilere sahip olmak faydalı olabilir.

### **Kaynaklar**

[c# “Mystery” of readonly — Programmer All](https://programmerall.com/article/1951107997/)

[Micro-optimization: the surprising inefficiency of readonly fields | Jon Skeet’s coding blog](https://codeblog.jonskeet.uk/2014/07/16/micro-optimization-the-surprising-inefficiency-of-readonly-fields/)

[Write safe and efficient C# code | Microsoft Docs](https://docs.microsoft.com/en-us/dotnet/csharp/write-safe-efficient-code#declare-immutable-structs-as-readonly)

[Performance traps of ref locals and ref returns in C# — Developer Support (microsoft.com)](https://devblogs.microsoft.com/premier-developer/performance-traps-of-ref-locals-and-ref-returns-in-c/)

[The ‘in’-modifier and the readonly structs in C# — Developer Support (microsoft.com)](https://devblogs.microsoft.com/premier-developer/the-in-modifier-and-the-readonly-structs-in-c/)

[Avoiding struct and readonly reference performance pitfalls with ErrorProne.NET — Developer Support (microsoft.com)](https://devblogs.microsoft.com/premier-developer/avoiding-struct-and-readonly-reference-performance-pitfalls-with-errorprone-net/)
