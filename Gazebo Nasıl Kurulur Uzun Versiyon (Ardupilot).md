Tekrardan merhabalar, bugün ardupilot için gazebo simülasyonu yapmayı deneyeceğiz. Hadi başlayalım. 


# Başlıyoruz.

Öncelikle konuya neden PX4-Autopilot sistemini direk kullanarak simülasyon yapmadığım açıklayayım. 
PX4-Autopilot sistemi direk pymavlink iletişim protokolü ile çalışmıyor. PX4 daha genel olarak MAVSDK ile çalışıyor. ancak gerçek hayatta kullanacağım kodda pymavlink iletişimini kullanmam gerekiyordu. Bunun içinde ayrıyeten ardupilot sistemini kurmam şart oldu. 

Şimdi bazı sorunların üstesinden gelelim.

# Sorunlar

Gazebo kurmak zaten kendi başına ayrı bir dert. Birde hiç bilmediğimiz bir ortamda bu gibi sistemleri çalıştırmak çok sıkıntı oluşturuyor. Şuanda yaşayacağımız en büyük sıkıntı gazebo ortamını kesintisiz çalıştırmak!

Gazebo sürekli olarak kendini yenilen bir simülasyon. Yeni sürümleri çok sade olmasına rağmen performans sorunları yaşatıyor. Eski gazebo ortamını daha çok seviyorum. Bunun için de eski gazebo ortamını kuracağım bu simülasyon ortamında. Sürümümüz genel hatları ubuntu20-04 için gazebo11, ubuntu18-04 için gazebo9 olacak. 

Bu simülasyonu windows ortamında wsl üzerinden ubuntu20-04 ile yapacağım. Buraya kadar her şey tamam. Performans sorununu eski versiyon ile kontrol altına alacağız. peki ama simülasyondaki modelleri nasıl çağıracağız? Burada PX4-Autopilot git reposundan yararlanacağız. kendisi de ubuntu için gazebo11 kurduğundan simülasyonu çalıştırırken sadece protokol değişikliği yapacağız. Ancak bu o kadar kolay olmayacak. PX4 kurulumunda (yani GitHub ortamındaki reposundan yüklediğimiz simülasyonu kastediyorum) her şey otomatik olarak bir bash ile çalışıyordu. şimdi ise her şeyi elle yapmamız gerekiyor. ama sanırım oradaki bash nasıl çalışıyorsa ufak düzeltmeler ile bizde kendi simülasyonumuza uyarlayabiliriz.  
Peki ya nasıl? İşte asıl soru bu ve en büyük sorunumuzda burada başlıyor zaten. Hadi o zaman yavaştan kuruluma geçelim. 

# Kurulum Kısmı

> [!info]
 Bu uzun versiyon yazısında her şeyi deneme yanılmaya göre yapacağım. Yani bu kurulum kısmı bu yazıda yaparken denediklerimi içerecek. Eğer buraları atlamak isterseniz kısa versiyona (normal kurulum) yazımıza bakabilirsiniz

