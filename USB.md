USB 為 Universal Serial Bus 的縮寫，在物理接口上有 Type A, Type B, Type C 三種，未來接口主要為 Type C。
![img](https://za3c.com.tw/wp-content/uploads/2022/01/Z10-3-1024x576.jpeg)

## 一、Type C 引腳
相較於另外兩種，Type C 可以正反插都能工作，除了上下各 12 個引腳為 180 度對稱，還有個 CC 引腳來做判斷。
![img](https://upload.wikimedia.org/wikipedia/commons/thumb/0/07/USB_Type-C_Receptacle_Pinout.svg/1280px-USB_Type-C_Receptacle_Pinout.svg.png)

Type C 主要靠腳位來同時支援 USB 2.0 與 USB 3.0，所以 3.0 更像是 2.0 的疊加，也因如此並不是所有引腳都會被做上去，會因功能而只保留某些引腳
1. 只能充電 6P：1, 4, 5, (8), 9, 12
2. 只支援 2.0 傳輸速度：1, 4, 5, 6, 7, (8), 9, 12

註：2.0 的傳輸在 TYPE C 主要是靠 D+/D-

## 二、USB 列舉
現在 USB 設備大多是隨插即用的，這與 USB 的協定有關。如果 TYPE C 中有 D+/D- 引腳，插入後就會被電腦偵測到。在 Windows/Linux/Mac 中都有個 USB Controller，OS 會一直去輪循每個 port 看看是否有新設備插入，每次間隔約 125 us，且只能由 OS 向設備發起詢問(除了喚醒外)。辨認設備主要靠**標示符**來辨認，每個 USB 設備中都包含一組標示符給電腦辨認
```C
#pragma pack(push, 1)
struct _DEVICE_DESCRIPTOR_STRUCT { 
    BYTE bLength;           //設備描述符的字節數大小，為0x12 
    BYTE bDescriptorType;   //描述符類型編號，為0x01 
    WORD bcdUSB;            //USB版本號 
    BYTE bDeviceClass;      //USB分配的設備類代碼，0x01~0xfe為標準設備類，0xff為廠商自定義類型 
                            //0x00不是在設備描述符中定義的，如 HID 
    BYTE bDeviceSubClass;   //usb分配的子類代碼，同上，值由USB規定和分配的 
    BYTE bDeviceProtocol;   //USB分配的設備協議代碼，同上 
    BYTE bMaxPacketSize0;   //端點0的最大包的大小 
    WORD idVendor;          //廠商編號 
    WORD idProduct;         //產品編號 
    WORD bcdDevice;         //設備出廠編號 
    BYTE iManufacturer;     //描述廠商字符串的索引 
    BYTE iProduct;          //描述產品字符串的索引 
    BYTE iSerialNumber;     //描述設備序列號字符串的索引 
    BYTE bNumConfiguration; //可能的配置數量 
}
#pragma pack(pop)
```
宣告此結構體時建議告訴編譯器不要對齊，否則在讀取描述符時容易錯誤。
