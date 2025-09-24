# Захват видеопотока из Unreal Engine

> как передать видеопоток из unreal engine 5 в другое приложение, например, в python скрипт? можно ли использовать для захвата такого потока opencv?

Да, можно передавать видеопоток из Unreal Engine 5 в Python/OpenCV. Есть несколько эффективных способов:

## 1. **Pixel Streaming (нативный способ UE5)**

### Настройка в UE5:
```cpp
// В Project Settings → Platforms → Pixel Streaming
// Включите Pixel Streaming
```

### Python клиент для Pixel Streaming:
```python
import asyncio
import websockets
import cv2
import numpy as np
import base64

class UE5PixelStreamClient:
    def __init__(self, uri="ws://localhost:80"):
        self.uri = uri
        self.frame_buffer = []
    
    async def receive_frames(self):
        async with websockets.connect(self.uri) as websocket:
            # Отправляем команду на подписку видео
            await websocket.send('{"type": "subscribe", "topic": "video"}')
            
            while True:
                message = await websocket.recv()
                data = json.loads(message)
                
                if data.get('type') == 'videoFrame':
                    # Декодируем base64 в изображение
                    img_data = base64.b64decode(data['data'])
                    nparr = np.frombuffer(img_data, np.uint8)
                    frame = cv2.imdecode(nparr, cv2.IMREAD_COLOR)
                    
                    # Обрабатываем кадр в OpenCV
                    self.process_frame(frame)
    
    def process_frame(self, frame):
        # Пример обработки OpenCV
        gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
        cv2.imshow('UE5 Stream', gray)
        cv2.waitKey(1)
```

## 2. **Render Target + Socket (рекомендуется)**

### C++ сторона UE5:
```cpp
// В .h файле
#include "Engine/TextureRenderTarget2D.h"
#include "Sockets.h"
#include "SocketSubsystem.h"

UCLASS()
class AStreamingActor : public AActor
{
    GENERATED_BODY()
    
public:
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    UTextureRenderTarget2D* RenderTarget;
    
    UPROPERTY(EditAnywhere)
    int32 StreamPort = 8888;
    
    virtual void BeginPlay() override;
    virtual void Tick(float DeltaTime) override;
    
private:
    FSocket* ClientSocket;
    bool CaptureAndSendFrame();
};

// В .cpp файле
void AStreamingActor::BeginPlay()
{
    Super::BeginPlay();
    
    // Создаем TCP сервер
    ISocketSubsystem* SocketSubsystem = ISocketSubsystem::Get(PLATFORM_SOCKETSUBSYSTEM);
    FSocket* ListenSocket = SocketSubsystem->CreateSocket(NAME_Stream, TEXT("StreamServer"));
    
    FIPv4Address Address(0, 0, 0, 0);
    FIPv4Endpoint Endpoint(Address, StreamPort);
    
    ListenSocket->Bind(Endpoint.ToInternetAddr().Get());
    ListenSocket->Listen(1);
    
    // Принимаем подключение
    ClientSocket = ListenSocket->Accept(TEXT("UE5StreamClient"));
}

bool AStreamingActor::CaptureAndSendFrame()
{
    if (!RenderTarget || !ClientSocket) return false;
    
    FTextureRenderTargetResource* RTResource = RenderTarget->GameThread_GetRenderTargetResource();
    if (!RTResource) return false;
    
    TArray<FColor> Bitmap;
    FRenderTarget* RenderTargetData = RTResource;
    
    FIntRect Rect(0, 0, RenderTarget->SizeX, RenderTarget->SizeY);
    RTResource->ReadPixels(Bitmap, FReadSurfaceDataFlags(), Rect);
    
    // Конвертируем в RGB данные
    TArray<uint8> RGBData;
    RGBData.SetNumUninitialized(RenderTarget->SizeX * RenderTarget->SizeY * 3);
    
    for (int32 y = 0; y < RenderTarget->SizeY; y++)
    {
        for (int32 x = 0; x < RenderTarget->SizeX; x++)
        {
            int32 Index = y * RenderTarget->SizeX + x;
            FColor Pixel = Bitmap[Index];
            
            int32 RGBIndex = Index * 3;
            RGBData[RGBIndex] = Pixel.R;
            RGBData[RGBIndex + 1] = Pixel.G;
            RGBData[RGBIndex + 2] = Pixel.B;
        }
    }
    
    // Отправляем размер данных
    int32 DataSize = RGBData.Num();
    ClientSocket->Send((uint8*)&DataSize, sizeof(DataSize));
    
    // Отправляем данные
    int32 BytesSent;
    ClientSocket->Send(RGBData.GetData(), DataSize, BytesSent);
    
    return true;
}

void AStreamingActor::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);
    CaptureAndSendFrame();
}
```

### Python клиент для OpenCV:
```python
import socket
import cv2
import numpy as np

class UE5VideoReceiver:
    def __init__(self, host='localhost', port=8888):
        self.host = host
        self.port = port
        self.socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.width = 1920  # Должно совпадать с RenderTarget SizeX
        self.height = 1080 # Должно совпадать с RenderTarget SizeY
        
    def connect_and_receive(self):
        self.socket.connect((self.host, self.port))
        print("Connected to UE5")
        
        try:
            while True:
                # Получаем размер данных
                size_data = self.socket.recv(4)
                if not size_data:
                    break
                    
                data_size = int.from_bytes(size_data, byteorder='little')
                
                # Получаем данные кадра
                received_data = b''
                while len(received_data) < data_size:
                    chunk = self.socket.recv(min(4096, data_size - len(received_data)))
                    if not chunk:
                        break
                    received_data += chunk
                
                if len(received_data) == data_size:
                    # Конвертируем в numpy array
                    frame_data = np.frombuffer(received_data, dtype=np.uint8)
                    frame = frame_data.reshape((self.height, self.width, 3))
                    
                    # Обрабатываем в OpenCV
                    self.process_frame(frame)
                    
        except Exception as e:
            print(f"Error: {e}")
        finally:
            self.socket.close()
    
    def process_frame(self, frame):
        # Пример обработки OpenCV
        # Конвертируем RGB в BGR (UE5 использует RGB, OpenCV - BGR)
        frame_bgr = cv2.cvtColor(frame, cv2.COLOR_RGB2BGR)
        
        # Применяем фильтры OpenCV
        blurred = cv2.GaussianBlur(frame_bgr, (15, 15), 0)
        edges = cv2.Canny(blurred, 100, 200)
        
        cv2.imshow('UE5 Stream - Original', frame_bgr)
        cv2.imshow('UE5 Stream - Edges', edges)
        
        if cv2.waitKey(1) & 0xFF == ord('q'):
            cv2.destroyAllWindows()
            return False
        return True

# Запуск
receiver = UE5VideoReceiver()
receiver.connect_and_receive()
```

## 3. **Shared Memory (самый быстрый способ)**

### C++ сторона UE5:
```cpp
// Создаем shared memory
FString SharedMemoryName = TEXT("UE5VideoStream");
HANDLE hMapFile = CreateFileMapping(
    INVALID_HANDLE_VALUE, NULL, PAGE_READWRITE, 0, 
    BUFFER_SIZE, *SharedMemoryName
);

void* pBuf = MapViewOfFile(hMapFile, FILE_MAP_ALL_ACCESS, 0, 0, BUFFER_SIZE);

// Записываем данные кадра в shared memory
memcpy(pBuf, RGBData.GetData(), RGBData.Num());
```

### Python клиент:
```python
import mmap
import numpy as np
import cv2

def read_shared_memory():
    with mmap.mmap(-1, 1920*1080*3, tagname='UE5VideoStream') as mm:
        while True:
            mm.seek(0)
            frame_data = np.frombuffer(mm.read(), dtype=np.uint8)
            frame = frame_data.reshape((1080, 1920, 3))
            
            # OpenCV обработка
            cv2.imshow('UE5 Shared Memory', frame)
            if cv2.waitKey(1) & 0xFF == ord('q'):
                break
```

## 4. **FFmpeg Pipe**

### UE5 сторона (через командную строку):
```cpp
// Запускаем FFmpeg как pipe
FString Command = TEXT("ffmpeg -f rawvideo -pixel_format rgb24 -video_size 1920x1080 -i - -f mpegts udp://localhost:1234");
```

### Python клиент:
```python
import subprocess
import cv2

pipe = subprocess.Popen([
    'ffmpeg', '-i', 'udp://localhost:1234',
    '-f', 'image2pipe', '-pix_fmt', 'bgr24', '-vcodec', 'rawvideo', '-'
], stdout=subprocess.PIPE, stderr=subprocess.PIPE)

while True:
    raw_frame = pipe.stdout.read(1920*1080*3)
    frame = np.frombuffer(raw_frame, dtype=np.uint8).reshape((1080, 1920, 3))
    cv2.imshow('UE5 FFmpeg Stream', frame)
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break
```

## Рекомендации:

1. **Для максимальной производительности**: Shared Memory
2. **Для простоты настройки**: Render Target + Socket  
3. **Для веб-приложений**: Pixel Streaming
4. **Оптимизируйте размер кадра** (например, 1280x720 вместо 4K)
5. **Используйте сжатие** при необходимости уменьшить трафик

OpenCV отлично подходит для приема и обработки видеопотока из UE5!
