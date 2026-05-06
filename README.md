# Windows Server Home Lab - Öğrenme ve Uygulama Günlüğü

Yüksek Erişilebilirlik (High Availability) ve Additional Domain Controller (ADC) Kurulumu

> *"Tek bir Domain Controller, sistem için 'Tek Nokta Hatası' (Single Point of Failure - SPOF) anlamına gelir. Bu aşamada, mimariyi yedekli çalışacak (Redundancy) ve yükü dağıtacak şekilde ikinci bir sunucuyla (ADC) genişleterek kurumsal süreklilik sağladım."*

**Görev:**
Linux KVM altyapısında Windows Server 2022 işletim sistemine geçiş yapmak, sanallaştırma sürücülerini (virtio) optimize etmek ve ortama ikinci bir sunucu dahil ederek Additional Domain Controller (ADC) ile Active/Active replikasyon ve DNS yük dengelemesi yapılandırmak.

---

### I. KVM Optimizasyonu ve Primary DC (Server 2022) İnşası
Sistem altyapısı en güncel sunucu işletim sistemine yükseltildi ve Linux ana makine ile tam uyum için gerekli optimizasyonlar sağlandı:

* **Virtio Entegrasyonu:** KVM üzerinde çalışan sanal makinelerde maksimum performans (disk/network I/O) ve ekran çözünürlüğü sorunlarını gidermek için `virtio-win` sürücüleri kuruldu.
* **Ağ ve Kimlik Yapılandırması:** 
    * Ana sunucu ismi `Sivas` olarak belirlendi.
    * Statik IP adresi `192.168.1.200` olarak atandı.
    * AD DS rolü kurularak "Sivas" sunucusu ortamın Primary Domain Controller'ı (PDC) haline getirildi.
* **Ön Hazırlık:** Replikasyon testleri için bu ana sunucu üzerinde çeşitli Test Kullanıcıları ve Organizasyonel Birimler (OU) oluşturuldu.

---

### II. Additional Domain Controller (ADC) Entegrasyonu
Ortamın yedekliliğini sağlamak adına ikinci bir Windows Server 2022 makinesi kurularak etki alanına dahil edildi:

1. **Rol Yapılandırması:** İkinci sunucuya AD DS rolü eklendi. Promote etme aşamasında "Add a new forest" yerine, ortamı genişletmek için **"Add a domain controller to an existing domain"** seçeneği tercih edildi.
2. **Kritik Servisler:** 
    * İkinci sunucunun isim çözümlemesi yapabilmesi ve dizin arama işlemlerini hızlandırması için **DNS Server** ve **Global Catalog (GC)** seçenekleri işaretli bırakıldı. 
    * Sunucu sadece okuma amaçlı (şube sunucusu) kullanılmayacağı için **RODC (Read-Only Domain Controller)** seçeneği işaretlenmedi. Tam yetkili bir DC olarak yapılandırıldı.
3. **Replikasyon Hedefi:** Kurulum sihirbazında "Replicate from" seçeneği altında ilk DC (`Sivas`) hedef gösterildi. Veritabanı (NTDS) ve SYSVOL klasör lokasyonları varsayılan standartlarda bırakıldı.

---

### III. Replikasyon ve Senkronizasyon Doğrulaması
İki sunucu arasındaki Multi-Master (Active/Active) çalışma yapısı test edildi:

* **Nesne Senkronizasyonu:** İlk DC'de oluşturulan OU ve kullanıcıların, kurulum tamamlandıktan hemen sonra ADC üzerine eksiksiz bir şekilde geldiği (replicate olduğu) görüldü.
* **Çift Yönlü İletişim:** İki makineden herhangi birinde yapılan bir değişikliğin diğerine anında yansıdığı doğrulandı. Bu yapı, hem istemci doğrulama yükünü paylaştırmak hem de donanımsal bir çökme anında sistemin kesintisiz çalışmasını (Fault Tolerance) sağlamak için kritiktir.
* **SYSVOL (GPO) Testi:** Makinelerin birinde oluşturulan yeni bir Group Policy Object (GPO) kuralının, diğer makinede "Yenile" (Refresh) işlemi yapıldığında listeye düştüğü görülerek politika senkronizasyonu onaylandı.

---

### IV. Sistem Yöneticisi Notu (RCA): DNS Yük Dengeleme ve "Cross-DNS" Mimarisi
* **Karşılaşılan Durum:** ADC yapılandırılırken, ağ ayarlarında DNS adresi olarak sadece ilk DC'nin IP adresi girilmişti. Bu durumda, ADC bir DNS sunucusu olmasına rağmen tüm sorguları ilk makineye yönlendiriyor, bu da yük dengelemesi (Load Balancing) yapılmasını engelliyordu.
* **Çözüm (Cross-DNS Yapılandırması):** Yedekli ve yük dağılımlı bir DNS mimarisi için bağdaştırıcı ayarları şu şekilde revize edildi:
    * **Primary DC (`Sivas`):** Preferred DNS: Kendi IP adresi (`127.0.0.1` veya yerel IP) | Alternate DNS: ADC'nin IP adresi.
    * **Additional DC:** Preferred DNS: Kendi IP adresi | Alternate DNS: Primary DC'nin (`Sivas`) IP adresi.
* **Sonuç:** Bu "Çapraz DNS" mantığı sayesinde her iki makine de ağa aktif olarak DNS hizmeti vermeye başladı. Bir sunucu kapandığında, istemciler "Alternate DNS" üzerinden diğer sunucuya giderek isim çözümlemeye sorunsuz devam edebilecektir.

---

### V. Topoloji Doğrulaması
* `Active Directory Users and Computers` (ADUC) yönetim konsolunda "Domain Controllers" OU'su incelendi.
* Ortamdaki her iki Server 2022 makinesinin de bu OU altında Global Catalog (GC) özellikli tam yetkili etki alanı denetleyicileri olarak listelendiği teyit edildi.

---
**Durum:** Phase 3 başarıyla tamamlandı. Ortam artık donanımsal arızalara karşı dirençli (Redundant) ve yükü dağıtabilen profesyonel bir mimariye kavuştu.
