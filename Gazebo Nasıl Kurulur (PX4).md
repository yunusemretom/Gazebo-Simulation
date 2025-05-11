Merhabalar tekrardan. Bugün ardupilot için gazebo nasıl kurulur bunu anlatacağım.
	 Bir güncelleme yapmam gerekiyor bu PX4 Mavsdk ile kontrol ediliyormuşi
# Temel Başlangıç
---

İlk olarak gazebo resmi sitesine gidip hangi ubuntu versiyonunu kullanmayı tercih ediyorsak ona uygun önerilen gazebo sürümünü indirelim. Ben bu anlatımı yaparken windowsta wsl kulanarak yapacağım tüm işlemleri. Wsl de kullanacağım sürüm ise Ubuntu-20.04 olacak.

Öncelikle komut istemini açalım ve ubuntu versiyonumuzu indirelim.

---

```powershell
wsl --install Ubuntu-20.04
# İndirilebilen ubuntu versiyonları öğrenmek için wsl --list --online ile bakabilirsiniz.
```

```powershell
sudo apt update
sudo apt upgrade -y # zorunlu değil ama sonrasında kütüphane hatası yaşamamk için önerilir.
```

Şimdi ise gazebo için [resmi sitesinden](https://gazebosim.org/docs/latest/getstarted/) adım adım indirmeleri yapalım.

---

[**PX4 Kaynak Kodunu İndirin**](https://docs.px4.io/main/en/dev_setup/building_px4.html) :

```powershell
git clone https://github.com/PX4/PX4-Autopilot.git --recursive
```

**Her şeyi kurmak için [ubuntu.sh](http://ubuntu.sh)'yi** hiçbir argüman vermeden (bir bash kabuğunda) çalıştırın :

```powershell
bash ./PX4-Autopilot/Tools/setup/ubuntu.sh
```

Kurulum bu şekilde tamamlanmış olur. Şimdi similasyonumuzu açalım.

---

```powershell
cd pat/to/PX4-Autopilot
make px4_sitl gazebo-classic
```

Bu şekilde ilk simülasyonumuz da çalışmış oluyor. Şimdi de bir kamera bağlayalım mı ne dersiniz?

---

```powershell
make px4_sitl gazebo_typhoon_h480
```

Videoyu Gstreamer Pipeline kullanarak görüntülemek mümkündür . Basitçe aşağıdaki terminal komutunu girin:

```powershell
gst-launch-1.0  -v udpsrc port=5600 caps='application/x-rtp, media=(string)video, clock-rate=(int)90000, encoding-name=(string)H264' ! rtph264depay ! avdec_h264 ! videoconvert ! autovideosink fps-update-interval=1000 sync=false
```

Bunlar basitçe simülasyonun ana temelleri. Eğer dökümantasyonu dikkatlıca okursanız bunlar herkesin yapabileceği sistemler. Lakin bizim bu projede yapmak istediklerimiz biraz farklı. Bizim yapmak istediğimz bir uçak simülasyonu yapmak ve ona kamera eklemek. Hadi gelin beraber yapmaya başlayalım 👨‍💻

# Detay Kısmı

---

Kamera verisi çekebiliyoruz. Evet ancak bir sorunumuz var. Bu videoyu opencv ile işleyemiyorum. Neden çünkü opencv GStream ile derlenmemiş oluyor genelde. Peki ne yapıcaz o zaman? Başka bir yol kullanıcaz. TCP ile alınan görüntüyü yayınlayıp OpenCV’de işleyeceğiz.

Yayın yapmak için şu komutu kullanabilirsiniz.

```powershell
gst-launch-1.0 -v udpsrc port=5600 caps="application/x-rtp, media=(string)video, clock-rate=(int)90000, encoding-name=(string)H264" ! rtph264depay ! avdec_h264 ! videoconvert ! jpegenc ! tcpserversink host=127.0.0.1 port=5000

```

Şimdi de pythondan okumaya başlayalım.

```python
import socket
import numpy as np
import cv2

# TCP Sunucusuna bağlan

TCP_IP = "127.0.0.1"
TCP_PORT = 5010
BUFFER_SIZE = 65536  # Büyük buffer, veri kaybını önler
  

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect((TCP_IP, TCP_PORT))

data = b""

while True:
    packet = sock.recv(BUFFER_SIZE)
    if not packet:
        break  # Bağlantı koptuğunda çık

    data += packet  # Paketi veri havuzuna ekle  

    # Eğer JPEG imzasını gördüysek işle

    start = data.find(b'\xff\xd8')  # Doğru JPEG başlangıç
    end = data.find(b'\xff\xd9')    # Doğru JPEG bitiş

  

    if start != -1 and end != -1:

        jpg = data[start:end+2]  # JPEG verisini al
        data = data[end+2:]  # Kullanılan veriyi at
 

        img = cv2.imdecode(np.frombuffer(jpg, dtype=np.uint8), cv2.IMREAD_COLOR)

        if img is not None:
            cv2.imshow("Stream", img)

        if cv2.waitKey(1) & 0xFF == ord("q"):
            break
    print("h")

sock.close()
cv2.destroyAllWindows()
```

### Kamera görüntüsü alabiliyoruz. Peki uçak modeline kamera nasıl ekleriz? Hadi yakından bakalım👇

---

Öncelikle kamerayı modelimize eklememiz gerek.

```python
cd ./PX4-Autopilot/Tools/simulation/gazebo-classic/sitl_gazebo-classic/models/plane
```

Bu yoldaki `plane.sdf` dosyasını biraz düzenliyeceğiz. Modele şu satırları eklemeniz gerekiyor. Bu şekilde uçağımızın üstünden görüntü alabileceğiz. Bu kodu `rudder_joint` elementinin altına koymuş bulunmaktayım.

```python
<link name="cgo3_camera_link">
      <inertial>
        <!-- place holder -->
        <pose>-0.041 0 -0.162 0 0 0</pose>
        <mass>0.1</mass>
        <inertia>
          <ixx>0.001</ixx>
          <ixy>0</ixy>
          <ixz>0</ixz>
          <iyy>0.001</iyy>
          <iyz>0</iyz>
          <izz>0.001</izz>
        </inertia>
      </inertial>
      <collision name='cgo3_camera_collision'>
        <pose>-0.041 0 -0.162 0 0 0</pose>
        <geometry>
          <sphere>
            <radius>0.035</radius>
          </sphere>
        </geometry>
        <surface>
          <friction>
            <ode>
              <mu>1</mu>
              <mu2>1</mu2>
            </ode>
          </friction>
          <contact>
            <ode>
              <kp>1e+8</kp>
              <kd>1</kd>
              <max_vel>0.01</max_vel>
              <min_depth>0.001</min_depth>
            </ode>
          </contact>
        </surface>
      </collision>
      <visual name='cgo3_camera_visual'>
        <pose>-0.05 0 0 0 0 3.141592</pose>
        <geometry>
          <mesh>
            <scale>0.001 0.001 0.001</scale>
            <uri>model://typhoon_h480/meshes/cgo3_camera_remeshed_v1.stl</uri>
          </mesh>
        </geometry>
        <material>
          <script>
            <name>Gazebo/DarkGrey</name>
            <uri>file://media/materials/scripts/gazebo.material</uri>
          </script>
        </material>
      </visual>
      <sensor name="camera_imu" type="imu">
        <always_on>1</always_on>
      </sensor>
      <sensor name="camera" type="camera">
        <pose>0.0 0 -0.162 0 0 0</pose>
        <camera>
          <horizontal_fov>2.0</horizontal_fov>
          <image>
            <format>R8G8B8</format>
            <width>640</width>
            <height>360</height>
          </image>
          <clip>
            <near>0.05</near>
            <far>15000</far>
          </clip>
        </camera>
        <always_on>1</always_on>
        <update_rate>20</update_rate>
        <visualize>true</visualize>
        <plugin name="GstCameraPlugin" filename="libgazebo_gst_camera_plugin.so">
            <robotNamespace></robotNamespace>
            <udpHost>127.0.0.1</udpHost>
            <udpPort>5600</udpPort>
        </plugin>
        <plugin name="CameraManagerPlugin" filename="libgazebo_camera_manager_plugin.so">
            <robotNamespace>typhoon_h480</robotNamespace>
            <interval>1</interval>
            <width>3840</width>
            <height>2160</height>
            <maximum_zoom>8.0</maximum_zoom>
            <video_uri>udp://127.0.0.1:5600</video_uri>
            <system_id>1</system_id>
            <cam_component_id>100</cam_component_id>
            <mavlink_cam_udp_port>14530</mavlink_cam_udp_port>
        </plugin>
      </sensor>
    </link>
    <joint name='cgo3_camera_joint' type='revolute'>
      <child>cgo3_camera_link</child>
      <parent>base_link</parent>
      <pose>-0.01 0.03 -0.162 0 0 0</pose>
      <axis>
        <xyz>0 -1 0</xyz>
        <limit>
          <lower>-1.5708</lower>
          <upper>0.7854</upper>
          <effort>100</effort>
          <velocity>-1</velocity>
        </limit>
        <dynamics>
          <damping>0.1</damping>
        </dynamics>
        <use_parent_model_frame>1</use_parent_model_frame>
      </axis>
      <physics>
        <ode>
          <implicit_spring_damper>1</implicit_spring_damper>
          <limit>
            <!-- testing soft limits -->
            <cfm>0.1</cfm>
            <erp>0.2</erp>
          </limit>
        </ode>
      </physics>
    </joint>
```

Kameranı yerini değiştirmek için yukardaki kodun şu kısmını değiştirebilirsiniz.

```powershell
<sensor name="camera" type="camera">
  <pose>0.0 0 0.262 0 0 0</pose> #bu kısımdaki pose değeri kameranın yerini verir.
  <camera>
```

Birde kameranın fps değerini değiştirmek için de kodun şu kısmını düzenleyebilirsiniz.

```powershell
</camera>
<always_on>1</always_on>
<update_rate>20</update_rate>#bu kısım kameranın anlık fps değerini arttırıp azaltan yer.
```

### Kameranın yayın yapıp yapmamasını kontrol etmek için de world dosyasını düzenlememiz gerekiyor. bunun için de şu adımları uygulayın.

---

```powershell
cd ./PX4-Autopilot/Tools/simulation/gazebo-classic/sitl_gazebo-classic/worlds
```

Bu klasöre gidip aşağıdaki kodu `world` etiketinin hemen altına yapıştırın.

```powershell
<gui>
	<plugin name="video_widget" filename="libgazebo_video_stream_widget.so"/>
</gui>
```

En sonda şöyle bir şey olmalı.

![image.png](attachment:f67a14ca-d6a1-495a-ae9b-5779c18ece60:image.png)

### Plane modelini kontrol etmek içiin QGroundController uygulaması gerek. Bunu indrimek için aşağıdaki adımları takip edebilirsiniz.

---

Öncelikle sistemimizi uygun hale getirelim.

```powershell
sudo usermod -a -G dialout $USER
sudo apt-get remove modemmanager -y
sudo apt install gstreamer1.0-plugins-bad gstreamer1.0-libav gstreamer1.0-gl -y
sudo apt install libfuse2 -y
sudo apt install libxcb-xinerama0 libxkbcommon-x11-0 libxcb-cursor-dev -y
```

Daha sonra uygulamayı indirilelim.

```powershell
wget <https://d176tv9ibo4jno.cloudfront.net/latest/QGroundControl.AppImage>
```

Ve izinleri ayarlayıp uygulamayı çalıştıralim.

```powershell
chmod +x ./QGroundControl.AppImage
./QGroundControl.AppImage 
```

<aside>

Şu hatayı verirse bunu yapın !

<aside> ⚠️

/tmp/.mount_QGroundX5Bwq/QGroundControl: error while loading shared libraries: libpulse-mainloop-glib.so.0: cannot open shared object file: No such file or directory

</aside>

```powershell
 sudo apt-get install libpulse-dev
```

</aside>

## Şimdi ise çoklu araç nasıl eklenir ona bakalım.

---

Yukarıda anlatılanlar tekli araç içindi çoklu araç için başka bir ayar yapmamız gerekiyor.

Öncelikle bu konuma gidin👇

```powershell
cd ./PX4-Autopilot/Tools/simulation/gazebo-classic/sitl_gazebo-classic/models/plane
```

Buradaki [`plane.sdf.ninja`](http://plane.sdf.ninja) dosyasına yine `rudder_joint` kısmına şu kodu ekleyin.

```powershell
<link name="cgo3_camera_link">
      <inertial>
        <!-- place holder -->
        <pose>-0.041 0 -0.562 0 0 0</pose>
        <mass>0.1</mass>
        <inertia>
          <ixx>0.001</ixx>
          <ixy>0</ixy>
          <ixz>0</ixz>
          <iyy>0.001</iyy>
          <iyz>0</iyz>
          <izz>0.001</izz>
        </inertia>
      </inertial>
      <collision name='cgo3_camera_collision'>
        <pose>-0.041 0 0.562 0 0 0</pose>
        <geometry>
          <sphere>
            <radius>0.035</radius>
          </sphere>
        </geometry>
        <surface>
          <friction>
            <ode>
              <mu>1</mu>
              <mu2>1</mu2>
            </ode>
          </friction>
          <contact>
            <ode>
              <kp>1e+8</kp>
              <kd>1</kd>
              <max_vel>0.01</max_vel>
              <min_depth>0.001</min_depth>
            </ode>
          </contact>
        </surface>
      </collision>
      <visual name='cgo3_camera_visual'>
        <pose>-0.05 0 0 0 0 3.141592</pose>
        <geometry>
          <mesh>
            <scale>0.001 0.001 0.001</scale>
            <uri>model://typhoon_h480/meshes/cgo3_camera_remeshed_v1.stl</uri>
          </mesh>
        </geometry>
        <material>
          <script>
            <name>Gazebo/DarkGrey</name>
            <uri>file://media/materials/scripts/gazebo.material</uri>
          </script>
        </material>
      </visual>
      <sensor name="camera_imu" type="imu">
        <always_on>1</always_on>
      </sensor>
      <sensor name="camera" type="camera">
        <pose>0.0 0 0.162 0 0 0</pose>
        <camera>
          <horizontal_fov>2.0</horizontal_fov>
          <image>
            <format>R8G8B8</format>
            <width>640</width>
            <height>360</height>
          </image>
          <clip>
            <near>0.05</near>
            <far>15000</far>
          </clip>
        </camera>
        <always_on>1</always_on>
        <update_rate>10</update_rate>
        <visualize>true</visualize>
        <plugin name="GstCameraPlugin" filename="libgazebo_gst_camera_plugin.so">
            <robotNamespace></robotNamespace>
            <udpHost>127.0.0.1</udpHost>
            <udpPort>{{ gst_udp_port }}</udpPort>
        </plugin>
        <plugin name="CameraManagerPlugin" filename="libgazebo_camera_manager_plugin.so">
            <robotNamespace>typhoon_h480</robotNamespace>
            <interval>1</interval>
            <width>3840</width>
            <height>2160</height>
            <maximum_zoom>8.0</maximum_zoom>
            <video_uri>{{ video_uri }}</video_uri>
            <system_id>{{ mavlink_id }}</system_id>
            <cam_component_id>{{ cam_component_id }}</cam_component_id>
            <mavlink_cam_udp_port>{{ mavlink_cam_udp_port }}</mavlink_cam_udp_port>
        </plugin>
      </sensor>
    </link>
    <joint name='cgo3_camera_joint' type='revolute'>
      <child>cgo3_camera_link</child>
      <parent>base_link</parent>
      <pose>-0.01 0.03 -0.162 0 0 0</pose>
      <axis>
        <xyz>0 -1 0</xyz>
        <limit>
          <lower>-1.5708</lower>
          <upper>0.7854</upper>
          <effort>100</effort>
          <velocity>-1</velocity>
        </limit>
        <dynamics>
          <damping>0.1</damping>
        </dynamics>
        <use_parent_model_frame>1</use_parent_model_frame>
      </axis>
      <physics>
        <ode>
          <implicit_spring_damper>1</implicit_spring_damper>
          <limit>
            <!-- testing soft limits -->
            <cfm>0.1</cfm>
            <erp>0.2</erp>
          </limit>
        </ode>
      </physics>
    </joint>
```

Ve daha sonra da şu kodu çalıştırın. Bu şekilde simülasyon çalışmış olmalı.

```powershell
./PX4-Autopilot/Tools/simulation/gazebo-classic/sitl_multiple_run.sh -m plane -n 3 
```

QGroundController’da komutları vererek uçakları havalandırabilirsiniz.

```powershell
./QGroundControl.AppImage 
```


Keyifli uçuşlar dileriz 🛫