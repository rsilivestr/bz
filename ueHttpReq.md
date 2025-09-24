# HTTP запросы в Unreal Engine

> как выполнять HTTP запросы в unreal engine? как получать поток данных (координат объектов) из удаленного сервера?

В Unreal Engine есть несколько способов выполнения HTTP запросов и работы с потоковыми данными. Рассмотрим основные подходы:

## 1. HTTP запросы через UE4/UE5

### Использование `FHttpModule` (C++)

```cpp
// В .h файле
#include "Interfaces/IHttpRequest.h"
#include "Interfaces/IHttpResponse.h"

// В .cpp файле
void AMyActor::MakeHttpRequest()
{
    // Создаем запрос
    TSharedRef<IHttpRequest> Request = FHttpModule::Get().CreateRequest();
    
    // Настраиваем запрос
    Request->SetURL("https://api.example.com/data");
    Request->SetVerb("GET");
    Request->SetHeader("Content-Type", "application/json");
    
    // Привязываем обработчики
    Request->OnProcessRequestComplete().BindUObject(this, &AMyActor::OnHttpResponseReceived);
    
    // Отправляем запрос
    Request->ProcessRequest();
}

void AMyActor::OnHttpResponseReceived(FHttpRequestPtr Request, FHttpResponsePtr Response, bool bWasSuccessful)
{
    if (bWasSuccessful && Response.IsValid())
    {
        FString ResponseString = Response->GetContentAsString();
        UE_LOG(LogTemp, Warning, TEXT("Response: %s"), *ResponseString);
        
        // Парсим JSON
        TSharedPtr<FJsonObject> JsonObject;
        TSharedRef<TJsonReader<>> Reader = TJsonReaderFactory<>::Create(ResponseString);
        
        if (FJsonSerializer::Deserialize(Reader, JsonObject) && JsonObject.IsValid())
        {
            // Обрабатываем данные
            float x = JsonObject->GetNumberField("x");
            float y = JsonObject->GetNumberField("y");
            float z = JsonObject->GetNumberField("z");
        }
    }
}
```

### Blueprint вариант
- Используйте node **HTTP Request** в Blueprint
- Доступен через правый клик → **HTTP Request**

## 2. WebSockets для потоковых данных

### Установка WebSocket плагина
1. Включите **WebSocket** плагин в Edit → Plugins
2. Добавьте в `.Build.cs`:
```cpp
PublicDependencyModuleNames.AddRange(new string[] {
    "WebSockets"
});
```

### Использование WebSockets (C++)

```cpp
// В .h файле
#include "IWebSocket.h"
#include "WebSocketsModule.h"

class AMyActor : public AActor
{
private:
    TSharedPtr<IWebSocket> WebSocket;
    
public:
    void ConnectWebSocket();
    void OnWebSocketMessage(const FString& Message);
    void OnWebSocketConnected();
    void OnWebSocketError(const FString& Error);
};

// В .cpp файле
void AMyActor::ConnectWebSocket()
{
    const FString ServerURL = "ws://localhost:8080/stream";
    const FString ServerProtocol = "ws";
    
    WebSocket = FWebSocketsModule::Get().CreateWebSocket(ServerURL, ServerProtocol);
    
    // Привязываем события
    WebSocket->OnConnected().AddUObject(this, &AMyActor::OnWebSocketConnected);
    WebSocket->OnMessage().AddUObject(this, &AMyActor::OnWebSocketMessage);
    WebSocket->OnConnectionError().AddUObject(this, &AMyActor::OnWebSocketError);
    
    // Подключаемся
    WebSocket->Connect();
}

void AMyActor::OnWebSocketMessage(const FString& Message)
{
    // Парсим потоковые данные (координаты объектов)
    TSharedPtr<FJsonObject> JsonObject;
    TSharedRef<TJsonReader<>> Reader = TJsonReaderFactory<>::Create(Message);
    
    if (FJsonSerializer::Deserialize(Reader, JsonObject))
    {
        FString ObjectId = JsonObject->GetStringField("id");
        float x = JsonObject->GetNumberField("x");
        float y = JsonObject->GetNumberField("y");
        float z = JsonObject->GetNumberField("z");
        
        // Обновляем позицию объекта в мире
        UpdateObjectPosition(ObjectId, FVector(x, y, z));
    }
}
```

## 3. TCP сокеты для низкоуровневых потоков

### TCP клиент (C++)

```cpp
// В .h файле
#include "Sockets.h"
#include "SocketSubsystem.h"

class AMyActor : public AActor
{
private:
    FSocket* Socket;
    bool bIsConnected;
    
public:
    void ConnectTCP();
    void ReceiveData();
    void ParseCoordinates(const FString& Data);
};

// В .cpp файле
void AMyActor::ConnectTCP()
{
    ISocketSubsystem* SocketSubsystem = ISocketSubsystem::Get(PLATFORM_SOCKETSUBSYSTEM);
    
    FString Host = "127.0.0.1";
    int32 Port = 9000;
    
    TSharedPtr<FInternetAddr> Addr = SocketSubsystem->CreateInternetAddr();
    bool bIsValid;
    Addr->SetIp(*Host, bIsValid);
    Addr->SetPort(Port);
    
    Socket = SocketSubsystem->CreateSocket(NAME_Stream, TEXT("TCPClient"));
    
    if (Socket->Connect(*Addr))
    {
        bIsConnected = true;
        // Запускаем асинхронное чтение
        FTimerHandle TimerHandle;
        GetWorldTimerManager().SetTimer(TimerHandle, this, &AMyActor::ReceiveData, 0.01f, true);
    }
}

void AMyActor::ReceiveData()
{
    if (!bIsConnected) return;
    
    uint32 PendingDataSize;
    if (Socket->HasPendingData(PendingDataSize))
    {
        TArray<uint8> ReceivedData;
        ReceivedData.SetNumUninitialized(PendingDataSize);
        
        int32 BytesRead;
        Socket->Recv(ReceivedData.GetData(), ReceivedData.Num(), BytesRead);
        
        FString ReceivedString = StringFromBinaryArray(ReceivedData);
        ParseCoordinates(ReceivedString);
    }
}
```

## 4. JSON парсинг для координат

```cpp
void AMyActor::ParseCoordinates(const FString& JsonString)
{
    TSharedPtr<FJsonObject> JsonObject;
    TSharedRef<TJsonReader<>> Reader = TJsonReaderFactory<>::Create(JsonString);
    
    if (FJsonSerializer::Deserialize(Reader, JsonObject))
    {
        // Одиночный объект
        if (JsonObject->HasField("x"))
        {
            FVector Coordinates;
            Coordinates.X = JsonObject->GetNumberField("x");
            Coordinates.Y = JsonObject->GetNumberField("y");
            Coordinates.Z = JsonObject->GetNumberField("z");
            
            UpdateObjectPosition("single", Coordinates);
        }
        // Массив объектов
        else if (JsonObject->HasField("objects"))
        {
            const TArray<TSharedPtr<FJsonValue>>* ObjectsArray;
            if (JsonObject->TryGetArrayField("objects", ObjectsArray))
            {
                for (const TSharedPtr<FJsonValue>& ObjectValue : *ObjectsArray)
                {
                    TSharedPtr<FJsonObject> ObjectData = ObjectValue->AsObject();
                    FString ObjectId = ObjectData->GetStringField("id");
                    FVector Coordinates;
                    Coordinates.X = ObjectData->GetNumberField("x");
                    Coordinates.Y = ObjectData->GetNumberField("y");
                    Coordinates.Z = ObjectData->GetNumberField("z");
                    
                    UpdateObjectPosition(ObjectId, Coordinates);
                }
            }
        }
    }
}
```

## 5. Blueprint реализация

### Для HTTP:
- **HTTP Request** node → обрабатывайте ответ в **On Success**

### Для WebSockets (через плагины):
- Установите плагин типа **WebSocket Blueprint**
- Используйте ноды **Connect**, **Send**, **On Message**

## Рекомендации:

1. **Для простых запросов** - используйте HTTP модуль
2. **Для потоковых данных в реальном времени** - WebSockets
3. **Для высокопроизводительных потоков** - TCP сокеты
4. **Всегда обрабатывайте ошибки** и таймауты
5. **Используйте асинхронные операции** чтобы не блокировать игровой поток

Пример JSON для координат:
```json
{
    "objects": [
        {"id": "player1", "x": 100.5, "y": 200.3, "z": 50.0},
        {"id": "npc1", "x": 150.0, "y": 180.7, "z": 50.0}
    ]
}
```
