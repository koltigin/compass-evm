# Compass-EVM

Bu akıllı sözleşme, keyfi işlem verileriyle başka bir akıllı sözleşmenin yürütülmesini mümkün kılmaktır.

Bu Vyper* ile yazılmıştır.
*Vyper, Ethereum Sanal Makinesi'ni (EVM) hedefleyen sözleşmeye dayalı, pitonik bir programlama dilidir. (https://vyper.readthedocs.io/en/stable/)

Kullanım Örneği:

* Paloma zincirinde hangi adresin Syntehtic Dai'yi alması gerektiğini belirterek Compass-EVM sözleşmesine 25 DAI gönderiyorsunuz.

* Paloma Blockchain'deki Doğrulayıcılar, Paloma zincirinde belirttiğiniz adres için 25 sentetik DAI basıldığını ve bunun gerçekleştiğini görüyor.

* 25 sentetik DAI'yi Paloma zincirinde Jim'e gönderiyorsunuz.

* Jim, sentetik Dai'yi Paloma blockchain'deki Turnstone modülüne gönderir ve hangi Ethereum adresinin onu alması gerektiğini belirtir.

* Paloma doğrulayıcıları Paloma Blockchain'deki sentetik DAI'yi yakar ve Ethereum'da Jim için 25 DAI'nin kilidini açar


## Güvenlik Modeli

Compass-EVM sözleşmesi temel olarak birkaç ince ayar içeren bir doğrulayıcı (validator) seti çoklu imzasıdır (multisig). Paloma üzerinde bir fikir birliği süreci ile kullanılmak üzere tasarlanmış olsa da, Compass-EVM sözleşmesi bu fikir birliği süreci hakkında hiçbir şey kodlamamaktadır. Burada iki ana işlem vardır;  update_valset, submit_logic_call.

* update_valset, çoklu imza (multisig) ile ilgili imzalayanları ve bunların göreceli güçlerini günceller. Bu, Paloma Blockchain üzerindeki doğrulayıcıyı (validator) yansıtır, böylece tüm Paloma doğrulayıcıları, paloma zincirindeki staking güçleri ile orantılı olarak imzalayıcılardır. Bir Update_Valset işlemi, kabul edilecek geçerli valset'in 2/3'ü tarafından imzalanmalıdır.

* submit_logic_call, başka bir akıllı sözleşmeye isteğe bağlı bir işlem göndermek için kullanılır. Bu mantık çağrısı (logic call) mevcut valet'in 2/3'ü tarafından imzalanmalıdır.

### update_valset

Bir valset, Validator'un Ethereum adreslerinin bir listesinden, oy kullanma gücünden ve tüm valset için bir kimlikten (ID) oluşur. update_valset yeni bir valset alır, mevcut valset ve mevcut valset'in yeni valset üzerindeki imzalarını alır. Sonra sağlanan mevcut valset'i kaydedilmiş kontrol noktasına karşı kontrol eder. Bu konu biraz açıklanma gerektirir. Valset'ler 100'den fazla doğrulayıcı içerdiğinden, bunların hepsini her seferinde Ethereum blok zincirinde saklamak oldukça maliyetli olacaktır. Bu nedenle, sadece mevcut valset'in bir karmasını saklıyoruz, ardından arayanın (caller) güncel adresleri, güçleri ve valset_id'yi tedarik etmesine izin verir.

Arayan tarafından sağlanan valset'in doğru olduğundan emin olduğumuzda, yeni valset kimliğinin geçerli valset ID (kimliğinden) daha yüksek olduğunu kontrol ediyoruz. Bu, (kimliklerin) ID'lerin çok düşük olduğu için eski valsetlerin gönderilmemesini sağlar. Not: Yeni Valset'ten kontrol ettiğimiz tek şey kimliktir. Yeni valset'in geri kalanı argümanlarda bu yönteme aktarılır, ancak yalnızca yeni valest'in kontrol noktasını yeniden oluşturur. Eğer biz kimliği (ID) kontrol etmeseydik, doğrudan kontrol noktasını geçmek mümkün olurdu.

Şimdi, make_checkPoint'i tekrar kullanarak gönderilen yeni valset'ten bir kontrol noktası oluşturuyoruz. Daha sonra bir kontrol noktası olarak kullanılmasının yanı sıra, önce mevcut valset'in yeni valset üzerindeki imzasını kontrol etmek için bir özet olarak kullanıyoruz. Bunu yapmak için check_validator_signatures kullanıyoruz.

check_validator_signatures bir valset, bir dizi imza, bir karma (hash) ve bir güç eşiği alır. Bu, eşiğe kadar eklenen karmayı (hash) imzalayan tüm doğrulayıcıların güçlerini kontrol eder.Biz, yeni valset'in mevcut valset'in en az 2/3'ü tarafından onaylandığını bu şekilde biliyoruz. Biz, aynı uzunlukta olması gereken mevcut valset ve imza dizisi üzerinde yineleme yapıyoruz. Tüm doğrulayıcılar (validator) için öncelikle imzanın sıfırlanıp sfırlanmadığını kontrol ediyoruz. Bu, belirli bir doğrulayıcının imzasını elde etmenin mümkün olmadığın anlamına gelir. Bu durumda, listedeki bir sonraki doğrulayıcıya geçiyoruz. İmzaların sadece 2/3'üne ihtiyacımız olduğu için her doğrulayıcının her seferinde imzalaması gerekli değildir ve onları es geçerek bir sonraki doğrulayıcıya geçmek herhangi bir doğrulayıcının çalışmayı bırakabilmesini durdurur.

Bir doğrulayıcı için bir imzamız varsa, bunu doğrularız. Eğer orada yanlış bir şey varsa bunu bir hata olarak kaydederiz. Ayrıca bir cumulative_power sayacını doğrulayıcının gücü ile artırıyoruz. Bir kere bu eşiği aştığında döngüden çıkıyoruz ve imzalar doğrulanmış oluyor! Eşik karşılanmadan döngü biterse, bir hata atıyoruz. Eşik karşılandıktan sonra döngüden çıkma şeklimiz nedeniyle, valset azalan güce göre sıralanırsa, genellikle imzaların çoğunluğunu değerlendirmeyi atlayabiliriz. Bu gaz tasarruflarından yararlanmak ve valsetlerin, validatörler tarafından azalan güç sırasına göre üretilmesi için önemlidir. Bu noktada, tüm kontroller tamamlandı ve şimdi valset'i güncelleme zamanı! Bu biraz antiklimaktiktir, çünkü yaptığımız tek şey yeni kontrol noktasını eskisi üzerinde kaydetmektir. Bir olay da yayınlanır (event emitted*).

*Bilgisayardaki her eylem bir olaydır. Eylemlere örnek olarak; bilgisayarın açılması, dosyanın açılması, dosyanın kapatılması, web sayfanın açılması, klavyeden bir tuşa basılması gibi örnekler verilebilir (https://www.yusufsezer.com.tr/node-js-events/).

### submit_logic_call

Compass-EVM, diğer akıllı sözleşmeye isteğe bağlı işlemi bu şekilde yürütür.

Update_valset'te yapılan aynı kontrollerden bazılarıyla başlarız, kontrol noktasına karşı sağlanan mevcut valset'i kontrol ederek. message_id'nin kullanılıp kullanılmadığını da kontrol ediyoruz. Bu, Compass-EVM tarafından ele alınan her ERC20 için bir ID (kimlik) saklar. Bu kimliğin amacı, eski mantık çağrısının (magic call) tekrar gönderilmemesini sağlamaktır. Paloma zincirinde, hiç gönderilmeyen ve kimliği artık gönderilemeyecek kadar düşük olan eski yığınları temizlemek için de kullanılır.

Mevcut doğrulayıcının imzalarını, yeni bir valset üzerinden imzalarını kontrol etmek için yukarıda kullanılan aynı yöntemi kullanarak mantık çağrısının (magic call) karma (hash) üzerindeki işaretlerini kontrol ediyoruz.

Şimdi hazırız. Mantık (logic) sözleşmesine isteğe bağlı işlemi yürütüyoruz.

Yük verileri 1024 byte'tan az olmalıdır.

## Olaylar

Her biri farklı bir amacı olan 2 farklı olay yayıyoruz. Biri, Paloma zinciri tarafından olayların düzensiz olmamasını sağlamak için kullanılan message_id adlı bir alan içerir. Bu, olaylardan biri her yayıldığında güncellenmelidir. Diğeri valset_id güncellendiğinde valset_id ve kontrol noktası yayar.

### LogicCallEvent

Bu, başarıyla işlenmiş bir logic_call işlemi hakkında bilgi içerir. message_id ve logic_contract adresini ve yük verilerini içerir. Paloma zinciri, bu bilgilerden işlemi tanımlayabilir.

### ValsetUpdated

Bu, valset her güncellendiğinde yayılır. Hiçbir zaman Paloma durumuna getirilmediği için _eventNonce öğesini içermez. Aktarıcılar tarafından send_logic_call veya update_valset'i çağırdıklarında kullanılır, böylece işleme doğru doğrulayıcı imzalarını dahil edebilirler.
