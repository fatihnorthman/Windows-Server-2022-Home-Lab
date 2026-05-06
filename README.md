# Windows Server Home Lab - Öğrenme ve Uygulama Günlüğü

## Phase 4: WSUS (Windows Server Update Services) Kurulumu ve Merkezi Yama Yönetimi

> *"Ağdaki tüm cihazların güncellemelerini tek tek internet üzerinden alması, bant genişliği israfı ve güvenlik zafiyetidir. Bu aşamada, şirket içi bir güncelleme otoritesi (WSUS) kurarak ağ trafiğini optimize ettim ve güncellemeler üzerinde merkezi kontrol sağladım."*

**Görev:**
Windows Server 2022 üzerinde WSUS rolünün kurularak yapılandırılması; etki alanına dahil edilen Windows 11 istemcisinin Microsoft sunucuları yerine yerel WSUS sunucusuna yönlendirilmesi (GPO ile) ve senkronizasyon/raporlama sorunlarının Kök Neden Analizi (RCA) ile giderilmesi.

---

### I. Ön Hazırlık ve İstemci Entegrasyonu
WSUS mimarisine geçmeden önce, güncellemeleri alacak olan test ortamı hazırlandı:
* Server 2022 (Primary DC) üzerinde Active Directory'de yeni bir test kullanıcısı oluşturuldu.
* Windows 11 istemci makinesi etki alanına (domain) dahil edildi ve oluşturulan kullanıcı ile oturum açılarak profilin oluşması sağlandı.

---

### II. WSUS Rolünün Kurulumu ve Veritabanı Mimarisi
Sunucu üzerinde WSUS rolü eklenirken, donanım kaynaklarına ve laboratuvar altyapısına uygun konfigürasyonlar tercih edildi:

1. **Rol Seçimi:** Server Manager üzerinden "Windows Server Update Services" rolü eklendi.
2. **Servis ve Veritabanı Yapılandırması:** 
    * **WID Connectivity (Windows Internal Database):** İşaretlendi. Ortamda harici bir SQL Server bulunmadığı için WSUS'un kendi dahili veritabanını kullanması sağlandı.
    * **WSUS Services:** İşaretlendi.
    * **SQL Server Connectivity:** Harici bir veritabanı olmadığı için bu seçenek devre dışı bırakıldı.
3. **Depolama Alanı:** Microsoft'tan çekilecek güncellemelerin yerel olarak barındırılacağı dizin `C:\WSUS` olarak belirlendi.
4. **Post-Deployment:** Kurulum tamamlandıktan sonra Server Manager üzerindeki bildirim paneline gidilerek "Post Deployment Configuration" (Kurulum Sonrası Yapılandırma) işlemi tetiklendi.

---

### III. WSUS Yapılandırma Sihirbazı (Configuration Wizard)
Kurulum sonrası `Tools > Windows Server Update Services` üzerinden yönetim konsolu (sivas paneli) açıldı ve şu parametreler girildi:

* **Upstream Server:** Kurulan makine ortamdaki ilk ve tek WSUS olduğu için "Synchronize from Microsoft Update" (Doğrudan Microsoft'tan çek) seçeneği tercih edildi.
* **Bağlantı ve Proxy:** İletişim portu olarak varsayılan **8530** kullanıldı. Ağda bir Proxy sunucusu olmadığı için bu adım boş geçildi ve Microsoft sunucularına bağlantı (Start Connecting) sağlandı.
* **Dil Seçenekleri:** Yalnızca İngilizce (EN) ve Türkçe (TR) güncellemelerin indirilmesi seçilerek disk tasarrufu sağlandı.
* **Ürün Seçimi (Products):** Sadece laboratuvar ortamında bulunan işletim sistemi olan "Windows 11" seçildi.
* **Kategori Seçimi (Classifications):** Updates, Upgrades, Security Updates, Critical Updates, Definition Updates ve Tools kategorileri işaretlendi.
* **Senkronizasyon Takvimi:** Laboratuvar ortamında anlık kontrolü elde tutmak için "Manual" (Elle senkronizasyon) seçildi. İşlemi kendi kontrolümde başlatmak için "Begin initial sync" (Başlangıç senkronizasyonunu başlat) kutucuğu boş bırakılarak sihirbaz tamamlandı.

---

### IV. Group Policy (GPO) ile İstemci Yönlendirmesi
**Durum Analizi:** WSUS kurulmuş olsa da, domaindeki istemcilerin (Client) bundan haberi yoktur ve güncellemeleri varsayılan olarak Microsoft'un internetteki sunucularından çekmeye çalışırlar. İstemcilere "Güncellemeleri internetten değil, içerideki şu sunucudan alacaksın" demek için GPO yapılandırıldı.

Group Policy Management üzerinden `WSUS-update` adında yeni bir GPO oluşturuldu ve aşağıdaki yapılandırmalar yapıldı:
*(Yol: Computer Configuration > Policies > Administrative Templates > Windows Components > Windows Update)*

1. **Specify an intranet Microsoft update service location:** `Enabled` yapıldı. Hem "Set the intranet update service for detecting updates" hem de "Set the intranet statistics server" kısımlarına WSUS adresi olan `http://192.168.1.200:8530` girildi.
2. **Configure Automatic Updates:** `Enabled` yapıldı. "4 - Auto download and schedule the install" seçilerek, her hafta Pazartesi günü saat 12:00'de indirilen güncellemelerin kurulması ayarlandı.
3. **Automatic Updates detection frequency:** `Enabled` yapıldı ve istemcinin sunucuya gidip yeni güncelleme olup olmadığını sorma sıklığı 12 saat olarak belirlendi.
4. **Do not connect to any Windows Update Internet locations:** `Enabled` yapıldı. Bu kritik kural ile makinenin dışarıdaki Microsoft Update sunucularına gitmesi tamamen yasaklandı.

GPO doğrudan etki alanına (Domain) bağlandı. İstemci makinesinde `gpupdate /force` komutu çalıştırılarak kuralların çekilmesi sağlandı. Ardından WSUS yönetim panelinden manuel senkronizasyon başlatıldı.

---

### V. Kök Neden Analizi (RCA) ve Sorun Giderme (Troubleshooting)

**Hata Belirtisi:** 
Senkronizasyonun bitmesine rağmen Windows 11 istemcisi, WSUS konsolundaki "Computers" sekmesine düşmedi. Ayrıca Windows 11'in kendi ayarlarında "Bazı ayarlar kuruluşunuz tarafından yönetilir" uyarısı görünmüyordu.

**Sorun İzolasyonu ve Test Süreci:**
1. **Bağlantı Testi (Port 8530):** 
    * İstemci tarayıcısından `http://192.168.1.200:8530/iuident.cab` adresine gidildi; `404 Not Found` hatası alındı.
    * Ancak `http://192.168.1.200:8530/ClientWebService/client.asmx` adresine gidildiğinde servisin çalıştığını gösteren "ClientWebServiceClient" WSDL bilgi ekranı başarıyla görüntülendi. Bu, sunucu tarafında IIS/WSUS servisinin ayakta olduğunu kanıtladı.
2. **Kayıt Defteri (Registry) Testi:** GPO'nun makineye gerçekten uygulanıp uygulanmadığını anlamak için CMD üzerinden şu komut çalıştırıldı:
    ```cmd
    reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate
    ```
    *Gelen Çıktı:*
    ```text
    WUServer    REG_SZ    [http://192.168.1.200:8530](http://192.168.1.200:8530)
    WUStatusServer    REG_SZ    [http://192.168.1.200:8530](http://192.168.1.200:8530)
    DoNotConnectToWindowsUpdateInternetLocations    REG_DWORD    0x1
    ```
    *Yorum:* Kayıt defteri çıktıları, GPO'nun makineye sorunsuz uygulandığını gösterdi. Hedef adres (192.168.1.200:8530) doğruydu.

**Kök Neden (Root Cause):**
İstemci tarafındaki Windows Update servisi (wuauserv) ve BITS servisi, eski güncelleme oturumlarını önbellekte (cache) tutmaktaydı. `SoftwareDistribution` klasöründeki eski yetkilendirme bilgileri (Authorization ID) sıfırlanmadığı için makine kendini yeni WSUS sunucusuna tanıtamıyordu.

**Çözüm Uygulaması:**
İstemci makinesinde komut satırı "Yönetici" (Administrator) olarak açılarak Windows Update servisleri durduruldu, önbellek temizlendi ve zorunlu tetikleme yapıldı:
```cmd
net stop wuauserv
net stop bits
rmdir /s /q C:\Windows\SoftwareDistribution
net start wuauserv
net start bits
wuauclt /resetauthorization /detectnow
UsoClient StartScan
