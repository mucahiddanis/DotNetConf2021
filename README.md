# DotNetConf2021

Modern Araçlarla .NET Core Uygulamaları Geliştirme (Bora Kaşmer)

[Youtube](https://www.youtube.com/watch?v=PzR1BdXpdRw&ab_channel=DevnotTV)

# Senaryo

Dövizle ürün satın alındığında doviz.com sitesinden doviz kurları pars edilerek TL karşılığına çevrilecek. Kur bilgisi alındığı zaman, ilgili client bilgilendirilecek (push notify). Kaydetme işlemi sırasında isteilen alanlar Encrypted olarak saklanacak. Güncelleme anında Audit Log tutulacak. Kur bilgisi belirlenmiş kayıtlar belli bir süre DB yerine belli bir süre memory cache'den getirilecek.

1. .Net Core 5.0 Back-End
 2. Angular 11 Front-End
 3. Go Web Parser Microservice
 4. RabbitMQ Queue
 5. ElasticSearch Audit Log
 6. SignalR Socket
 7. Redis Distributed Cache

# Kullanılan Teknolojiler

 

 - ElasticSearch
 - Kibana (ElasticSearch monitoring)
 - Redis
 - Redis Desktop Manager
 - RabbitMQ
 - AutoMapper
 - GoLang
 - SignalR
 - .Net Core
 - MS SQL
 - Angular

# SÜREÇ

### Her Alım İçin Parse İşlemi

Kaydetme işlemi en hızlı şekilde yapılmalı ve parse işlem yapılırken, kullanıcı sayfa üzerinde diğer işlemlere devam edebilmelidir. Kur bilgisi çekilip kaydedildikten sonra, performans amaçlı belli bir süre ilgili ürün için DB'ye gidilmemelidir.

Parse işlemi asenkron yürütülecektir. Her kaydetme işlemi kur değeri bulunmadan yapılacak ve ilgili ürün, RabbtMQ'ya atılacaktır. Go Mcroserivce, Queue'dan aldığı ürünün kur bilgisini, dövz.com'dan parse edip, .NET Core servise Post edecektir. Kur bilgisi bulunan kayıt, Redis'e de atılacaktır.

## DB, Core, Controller, Service, Repository 

Öncelikle proje katmanlara ayrılır. DB Firsti le DAL katmanı oluşturulur. Endpointler Controller katmanında istekleri karşılar. Service katmanında tüm business işlemleri yapılır. DB ile alakalı tüm işler Repostory katmanında halledilir. Ayrıca Entity işaretlenmş se, Loglama ve DataEncrypton Global olarak bu katmanda Service Injecton ile yapılır. Ortak kullanılan araç ve modeller Core katmanında saklanır.

## Giriş - Gelişme - Sonuç

Kod okunaklığının artması, Test ve Debug İşlemlerinin hızlanması, birden fazla kişinin aynı projede çalışması için, Katmanlı Mimari büyük kolaylıktır.

**Katman 1 (Controller):** Güvenlik amaçlı, mobile ve web servisler burada karşılanır. SignalR tetiklenir. RabbtMQ'ya işlenmesi amacı ile, kayıt atılır. Yani bir çeşit Adaptordür.

**Katman 2 (Service):** Tüm sorguların ve hesaplamaların yapıldığı yerdir. İlgili kayıdın, Redisde olup olmadığına burada bakılır.

**Katman 3 (Repository):** DB'ye yazma ve okuma işleri, Encrypton ve Elastic Search'e Loglama, global olarak burada yapılır.


## Mikroservis İşlemler

HER KAYIT İŞLEMİNDE "DOVIZ.COM"'UN PARSE İŞLEMİ UZUN SÜRECEĞİNDEN, GO MİCROSERVİS YARDIMI İLE, ARKADA PARS İŞLEMİ ASENKRON OLARAK YAPILIR VE SONUÇ .NET CORE SERVİSİNE POST EDİLİR.

> Insert olan product kaydı, Db'ye ve Redis'e kaydedildikten sonra, vakit alan pars işleminin asenkron yapılması için, RabbtMQ'ya atılır. Entity'e ait kritik kolonlar, şifrelenerek atılır. Go servisinde "Product" channel'ından alınan kayıt, "dövz.com" pars edilerek güncellenir. Ve .Net Core servisine ürünün son güncel TL fiyatlı hali, Post edilir

## Ürünün Güncellenmesi

Go Microservice'den gelen güncel TL fiyatlı ürün bilgisi, Controller katmanından servis katmanına gönderilirken, aynı zamanda tüm clientlara signalR kullanılarak Real Time post edilir.

 - Güncellenen ürün, signalR ile tüm clientlara Real Time gönderilir
 - Servis katmanında ürünün son toplam TL fiyatı hesaplanıp, Repository katmanında DB'ye kaydedilir. Ayrıca hassas datalar, değişmiş ise şifrelenerek atılır.
 - Servis katmanında, DB'de güncelenen ürün bilgisi, Redis'de de güncellenir.
 - Repository katmanında, güncellenen ürünün Audit Log kaydı, ElasticSearch'e indexlenir.


> Repository katmanında, güncellenen Entityler içinden sadece, **"IAuditable"** Interface'nden türeyenler, ElasticSearch'e indexlenr. Ve Entity üzerinde sadece **"CryptoData"** ile şaretli kolonlar, şifrelenerek DB'ye atılır.



### KAYIT SONRADAN GÜNCELLENİR İSE NE OLUR? 

Servis katmanında, Redis güncellenir. Repository katmanında DB güncellenir ve ElasticSearch'e Audit Index atılır. Socket ile diğer clientlara push notify yapılmaz. O işlem sadece Kur bilgisi parse edilince, anlık yapılır.


### NEDEN MİCROSERVİS OLARAK GO KULLANILDI? 

Projenin başta çok hızlı ayağa kalkması, zamanla harcadığı bellek miktarının çok az olması, bir sitenin Pars edilebilmesi için çok pratik bir kütüphanesinin olması ve stabilizasyonu için tercih edilmiştir.

