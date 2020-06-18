# 2. Протокол клиентского API

Клиентский API используется для взаимодействия клиентов и хабов.

## 2.1 Получение списка хабов
### 2.1.1 GetNodesRequest
Чтобы пользователь смог выбрать хаб, к которому ему нужно подключиться, необходимо получить список доступных хабов.
Для это требуется установить WebSocket-соединение с лицензиаром и отправить запрос на получение списка хабов.

| Название      | Тип значения    | Описание                                                                                                                                            |
|---------------|-----------------|-----------------------------------------------------------------------------------------------------------------------------------------------------|
| RequestType   | Unsigned Int8   | Тип запроса. Значение: 1                                                                                                                   |
| RequestId     | Int64           | Идентификатор запроса                                                                                                                               |
| Type          | Unsigned Int8   | Тип объекта, унаследовано от CommunicationObject. Принимает значение 6                                                                              |
| NodeId        | Int64           | Идентификатор хаба для постраничной навигации                                                                                                       |
| SearchQuery   | String          | Строка для поиска хабов по тэгу или по имени                                                                                                          |
| NodesIds      | Int64[]         | Список идентификаторов хабов для получения информации о конкретных хабах                                                                              |
| NodePublicKey | Unsigned Int8[] | Публичный ключ хаба для получения информации о скрытом хабе.  Если запрос отправляется клиентом, то ключ передается в виде строки в формате Base64. |
### 2.1.2 NodesResponse
На запрос получения списка хабов придет ответ следующего формата:
| Название | Тип значения | Описание |
| -| -| -| 
| Nodes | [Node](APIObjects.md?#45-node)[] |  Список объектов с информацией о хабах |
| RequestId | Int64 | Идентификатор запроса  |
| ResponseType | Int8 | Тип ответа. Значение: 2 |
| Type | Type | Тип объекта. Значение: 7 |
## 2.2 Подключение к хабу
1. Из полученного ранее списка хабов необходимо выбрать нужный хаб.
2. Из поля ` Node.Domains ` необходимо выбрать первый элемент.
3. Далее нужно подключиться по адресу: 
   `wss://{Node.Domains[0]}:{Node.ClientsPort}/`. Например:
   `wss://testnode1-1.ymess.org:5000/`
### 2.2.1 Создание зашифрованного соединения.
По умолчанию устанавливается обычное SSL-соединение. Для большей безопасности существует возможность установки дополнительного уровня шифрования. Для этого необходимо отправить на хаб запрос 
`SetConnectionEncryptedRequest`
### SetConnectionEncryptedRequest
| Название | Тип значения | Описание |
| -| -| -| 
| PublicKey | String | Публичный ключ шифрования в формате Base64 |
| SignPublicKey | String | Публичный ключ подписи в формате Base64  |
| UserId | Int64 | Идентификатор пользователя |
| PublicKeyId | Int64 | Идентификатор публичного ключа шифрования |
| SignPublicKeyId | Int64 | Идентификатор публичного ключа подписи |
| NodeId | Int64 | Идентификатор хаба |
| RequestType | RequestType | Тип запроса. Значение: 255 |

При успешной обработке запроса установится зашифрованное соединение. Ответ придет в незашифрованном виде, но все последующие запросы/ответы/уведомления должны быть зашифрованы симметричным ключом, который вернул хаб.

Ответ на запрос: [EncryptedKeyResponse](#2117-encryptedkeyresponse)
## 2.3 Запрос на верификацию
Для регистрации, авторизации, смены номера телефона/email-адреса требуется предварительно отправить запрос на верификацию, чтобы получить код подтверждения.
### 2.3.1 VerificationUserRequest
| Название | Тип значения | Описание |
| -| -| -| 
| IsRegistration | Boolean | Отправляется ли запрос перед регистрацией |
| VerificationType | VerificationType | Тип верификации |
| Uid | String | Данные, идентифицирующие пользователя |
| NodeId | Int64 | Идентификатор хаба |
| RequestType | RequestType | Тип запроса. Значение: 42 |

Ответ на запрос:  [ResultResponse](#2110-resultresponse)
### 2.3.2 VerificationType
| Название | Значение | Описание                                        |
|----------|----------|-------------------------------------------------|
| Phone    | 0        | В поле Uid находится номер телефона             |
| Email    | 1        | В поле Uid находится email                      |


## 2.4 Регистрация пользователей
В зависимости от значения `Node.RegistrationMethod` клиент должен отправить объект `User` с заполненным полем `Phones` или с заполненнным полем `Emails`. Перед отправкой запроса на регистрацию, клиент должен отправить запрос на верификацию электронной почты или номера телефона, в зависимости от значения `Node.RegistrationMethod`, и получить код верификации. Если хаб не требует ни номер телефона, ни email, запрос на верификацию отправлять не требуется.
### 2.4.1 NewUserRequest
| Название | Тип значения | Описание |
| -| -| -| 
| User | User | Объект, содержащий информацию о пользователе |
| VCode | Int16 | Код верификации |
| RequestType | RequestType | Тип запроса. Значение: 37 |

Ответ на запрос: [TokensResponse](#21122-tokensresponse)
## 2.5 Авторизация пользователей
Авторизоваться на хабе можно несколькими способами:
* [Запросить код верификации](#231-verificationuserrequest) и [отправить запрос `LoginRequest`](#251-loginrequest) c заполненными полями `Uid` и `VCode`.
* [Отправить запрос `LoginRequest`](#251-loginrequest) c заполненным полем `Token.AccessToken` и `Token.UserId`. Чтобы запрос выполнился успешно, необходимо, чтобы значение `Token.AccessTokenExpirationTime` было больше текущего значения unix-времени. В ином случае необходимо заполнить поле `Token.RefreshToken`. Если значение `Token.RefreshTokenExpiration` меньше текущего unix-времени, следует авторизовываться с помощью способа 1 или 3.
* Авторизация с помощью QR-кода. Необходимо [отправить запрос `CheckQRCodeRequest`](#254-checkqrcoderequest)
### 2.5.1 LoginRequest
| Название | Тип значения | Описание |
| -| -| -| 
| Uid | String | Данные, идентифицирующие пользователя |
| Token | Token | Токены пользователя |
| VCode | Int16 | Код верификации |
| LoginType | LoginType | Способ авторизации пользователя |
| UidType | UidType | Тип данных, находящихся в поле Uid |
| DeviceTokenId | String | Firebase-токен устройства для отправки push-уведомлений |
| NodeId | Int64 | Идентификатор хаба |
| DeviceName | String | Название устройства |
| OSName | String | Название ОС |
| AppName | String | Название приложения |
| RequestType | RequestType | Тип запроса. Значение: 33 |

Ответ на запрос: [TokensResponse](#21122-tokensresponse)
### 2.5.2 LoginType
| Название         | Значение | Описание                               |
|------------------|----------|----------------------------------------|
| AccessToken      | 0        | Авторизация с помощью access-токена    |
| VerificationCode | 1        | Авторизация с помощью кода верификации |
### 2.5.3 UidType
| Название | Значение | Описание                            |
|----------|----------|-------------------------------------|
| Phone    | 0        | В поле Uid находится номер телефона |
| Email    | 2        | В поле Uid находится email          |

### 2.5.4 CheckQRCodeRequest
| Название | Тип значения | Описание |
| -| -| -| 
| QR | QRCodeContent | Объект, содержащий информацию, хранящуюся в QR-коде |
| DeviceTokenId | String | Firebase-токен устройства для отправки push-уведомлений |
| DeviceName | String | Название устройства |
| OSName | String | Название ОС |
| AppName | String | Название приложения |
| RequestType | RequestType | Тип запроса. Значение: 75 |

## 2.6 Сообщения
См. [Message](APIObjects.md#46-message)
### 2.6.1 Отправка и получение сообщений
Для того, чтобы отправить сообщение, необходимо:
1. Выбрать получателя сообщения (заполнить поле `Message.ReceiverId`, если сообщение предназначено конкретному пользователю, или заполнить поля `Message.ConversationId` и `Message.ConversationType`, если сообщение отправляется в чат или канал)
 2. Заполнить поле `Message.Text` или `Message.Attachments`, или и то и другое
 3. Отправить запрос `SendMessagesRequest`, поместив инициализированный ранее объект `Message` в массив `SendMessagesRequest.Messages`. Запрос позволяет отправлять сразу несколько сообщений в разные чаты.
#### 2.6.1.1 SendMessagesRequest
 Запрос на отправку сообщений

| Название | Тип значения | Описание |
| -| -| -| 
| Messages | Message[] | Список сообщений |
| RequestType | RequestType | Тип запроса. Значение: 39 |


#### 2.6.1.2 GetMessagesRequest
 Запрос на получение сообщений из беседы

| Название | Тип значения | Описание |
| -| -| -| 
| ConversationType | ConversationType | Тип беседы |
| ConversationId | Int64 | Идентификатор беседы |
| NavigationMessageId | UUID | Идентификатор сообщения для постраничной навигации |
| Direction | Boolean | Направление навигации |
| MessagesId | UUID[] | Список идентификаторов сообщений для получения конретных сообщений |
| AttachmentsTypes | Int8[] | Список типов вложений для получения сообщений с вложениями конкретных типов |
| RequestType | RequestType | Тип запроса. Значение: 25 |

#### 2.6.1.3 SearchMessagesRequest
 Запрос на поиск сообщений

| Название | Тип значения | Описание |
| -| -| -| 
| ConversationType | ConversationType | Тип беседы, указывается для поиска сообщений в конкретной беседе |
| ConversationId | Int64 | Идентификатор беседы, указывается для поиска сообщений в конкретной беседе |
| Query | String | Поисковый запрос |
| NavigationMessageId | UUID | Идентификатор сообщения для постраничной навигации |
| NavigationConversationType | ConversationType | Тип беседы для постраничной навигации |
| NavigationConversationId | Int64 | Идентификатор беседы для постраничной навигации |
| RequestType | RequestType | Тип запроса. Значение: 72 |

Ответ на вышеперечисленные запросы: [MessagesResponse](#21113-messagesresponse)
#### 2.6.1.4 DeleteMessagesRequest
Запрос на удаление сообщений

| Название | Тип значения | Описание |
| -| -| -| 
| MessagesIds | UUID[] | Идентификаторы удаляемых сообщений |
| ConversationId | Int64 | Идентификатор беседы |
| ConversationType | ConversationType | Тип беседы |
| RequestType | RequestType | Тип запроса. Значение: 43 |

Ответ на запрос: [UpdatedMessagesResponse](#21123-updatedmessagesresponse)

#### 2.6.1.5 MessagesReadRequest
Запрос на прочтение сообщений.

| Название | Тип значения | Описание |
| -| -| -| 
| MessagesId | UUID[] | Идентификаторы сообщений |
| ConversationType | ConversationType | Тип беседы |
| ConversationId | Int64 | Идентификатор беседы |
| RequestType | RequestType | Тип запроса. Значение: 35 |

Ответ на запрос: [ResultResponse](#2110-resultresponse)

#### 2.6.1.6 ConversationActionRequest
Запрос на оповещение других участников беседы о действии, совершаемом пользователем

| Название | Тип значения | Описание |
| -| -| -| 
| ConversationId | Int64 | Идентификатор беседы |
| ConversationType | ConversationType | Тип беседы |
| Action | ConversationAction | Тип действия |
| RequestType | RequestType | Тип запроса. Значение: 79 |

#### 2.6.1.7 ConversationAction
| Название         | Значение | Описание                    |
|------------------|----------|-----------------------------|
| TypingText       | 0        | Набор текста                |
| RecordingVoice   | 1        | Запись голосового сообщения |
| RecordingVideo   | 2        | Запись видеосообщения       |
| UploadingFile    | 3        | Загрузка файла              |
| UploadingPicture | 4        | Загрузка изображения        |
| UploadingAudio   | 5        | Загрузка аудио              |
| UploadingVideo   | 6        | Загрузка видео              |
| CreatingPoll     | 7        | Создание опроса             |
| Screenshot       | 8        | Скриншот                    |

#### 2.6.1.8 Призыв пользователя в сообщении
 Если сообщение содержит текст следующего вида:
`[UserCalling](userId=<value>)`
где <value> идентификатор пользователя, пользователю придет push- или
WebSocket-уведомление в виде объекта `NewMessageNotice`, в котором поле `Call` будет равно `TRUE`

### 2.6.2 Вложения
 Для отправки сообщения с вложением необходимо [сформировать объект вложения](APIObjects.md#461-attachment) и поместить его в массив `Message.Attachments`
### 2.6.3 Обмен зашифрованными сообщениями
На данный момент отправка оконечно зашифрованных сообщений возможна только в диалогах с двумя участниками. Для отправки зашифрованного сообщения необходимо:
 1. Отправить специальное сообщение, содержащее [вложение с асимметрично зашифрованным симметричным ключом](APIObjects.md#464-keymessage):  
     1. Необходимо получить публичный ключ шифрования собеседника с помощью запроса `GetUserPublicKeysRequest`. Предполагается, что клиент [отправляет на хаб новые ключи](#2632-addkeysrequest) с определенной периодичностью.
     2. Сгенерировать симметричный ключ и зашифровать его полученным ранее публичным ключом
     3. После успешной отправки сообщения с зашифрованным симметричным ключом этот ключ можно использовать для шифрования контента
 2. Сформировать вложение с [зашифрованным сообщением](APIObjects.md#463-encryptedmessage) и отправить собеседнику
#### 2.6.3.1 GetUserPublicKeysRequest
Запрос на получение публичных ключей

| Название | Тип значения | Описание |
| -| -| -| 
| UserId | Int64 | Идентификатор пользователя |
| NavigationTime | Int64 | Время генерации ключа для постраничной навигации |
| Direction | Boolean | Направление навигации |
| KeysId | Int64[] | Идентификаторы ключей, если необходимо получить конкретные |
| RequestType | RequestType | Тип запроса. Значение: 105 |

Ответ на запрос: [KeysResponse](#21112-keysresponse)
#### 2.6.3.2 AddKeysRequest
Запрос на добавление ключей пользователя

| Название | Тип значения | Описание |
| -| -| -| 
| Keys | Key[] | Ключи |
| RequestType | RequestType | Тип запроса. Значение: 100 |

Ответ на запрос: [KeysResponse](#21112-keysresponse)

### 2.6.4 Чаты
См. [Chat](APIObjects.md#42-chat)
#### 2.6.4.1 NewChatsRequest
Запрос на создание чата

| Название | Тип значения | Описание |
| -| -| -| 
| Chats | Chat[] | Список чатов |
| RequestType | RequestType | Тип запроса. Значение: 36 |

Ответ на запрос: [ChatsResponse](#2113-chatsresponse)
#### 2.6.4.2 EditChatsRequest
Запрос на редактирование инфорации о чате. См. [EditChat](APIObjects.md#422-editchat)

| Название | Тип значения | Описание |
| -| -| -| 
| Chats | EditChat[] | Список редактируемых чатов |
| RequestType | RequestType | Тип запроса. Значение: 9 |

Ответ на запрос: [ChatsResponse](#2113-chatsresponse)
#### 2.6.4.3 AddUsersToChatsRequest
Запрос на добавление пользователей в чаты.

| Название | Тип значения | Описание |
| -| -| -| 
| ChatsId | Int64[] | Идентификаторы чатов |
| UsersId | Int64[] | Идентификаторы пользователей |
| RequestType | RequestType | Тип запроса. Значение: 1 |

Ответ на запрос: [ChatUsersResponse](#2114-chatusersresponse)
#### 2.6.4.4 EditChatUsersRequest
Запрос на редактирование пользователей чата. См. [ChatUser](APIObjects.md#421-chatuser)

У пользователей чатов могут быть различные полномочия. В зависимости от них пользователи могут совершать различные действия в чате.
 
 * Обычный участник чата может только отправлять сообщения
 * Модератор может блокировать обычных пользователей
 * Администратор может назначать/удалять модераторов, редактировать информацию о чате
 * Создатель чата может назначать администраторов, удалить чат

| Название | Тип значения | Описание |
| -| -| -| 
| ChatUsers | ChatUser[] | Список отредактированных пользователей чата |
| ChatId | Int64 | Идентификатор чата |
| RequestType | RequestType | Тип запроса. Значение: 3 |

Ответ на запрос: [ChatUsersResponse](#2114-chatusersresponse)
#### 2.6.4.5 GetChatUsersRequest
Запрос на получение списка пользователей чата

| Название | Тип значения | Описание |
| -| -| -| 
| ChatId | Int64 | Идентификатор чата |
| NavigationUserId | Int64 | Идентификатор пользователя для постраничной навигации |
| RequestType | RequestType | Тип запроса. Значение: 17 |

Ответ на запрос: [ChatUsersResponse](#2114-chatusersresponse)
#### 2.6.4.6 GetChatsRequest
Запрос на получение информации о чатах

| Название | Тип значения | Описание |
| -| -| -| 
| ChatsId | Int64[] | Идентификаторы чатов |
| RequestType | RequestType | Тип запроса. Значение: 17 |

Ответ на запрос: [ChatsResponse](#2113-chatsresponse)

### 2.6.5 Каналы
#### 2.6.5.1 CreateChannelRequest
Запрос на создание канала. См. [Channel](APIObjects.md#43-channel)

| Название | Тип значения | Описание |
| -| -| -| 
| Channel | Channel | Канал |
| Subscribers | ChannelUser[] | Приглашенные пользователи |
| RequestType | RequestType | Тип запроса. Значение: 47 |

Ответ на запрос: [ChannelsResponse](#2111-channelsresponse)
#### 2.6.5.2 GetChannelsRequest
Запрос на получение каналов

| Название | Тип значения | Описание |
| -| -| -| 
| ChannelsId | Int64[] | Идентификаторы каналов |
| RequestType | RequestType | Тип запроса. Значение: 52 |

Ответ на запрос: [ChannelsResponse](#2111-channelsresponse)

#### 2.6.5.3 EditChannelRequest
Запрос на редактирование канала

| Название | Тип значения | Описание |
| -| -| -| 
| Channel | Channel | Канал |
| RequestType | RequestType | Тип запроса. Значение: 49 |

Ответ на запрос: [ChannelsResponse](#2111-channelsresponse)
#### 2.6.5.4 AddUsersToChannelsRequest
Запрос на добавление пользователей в каналы. См. [ChannelUser](APIObjects.md#431-channeluser)

| Название | Тип значения | Описание |
| -| -| -| 
| ChannelsId | Int64[] | Идентификаторы каналов |
| UsersId | Int64[] | Идентификаторы пользователей |
| RequestType | RequestType | Тип запроса. Значение: 48 |

Ответ на запрос: [ChannelUsersResponse](#2112-channelusersresponse)

#### 2.6.5.5 EditChannelUsersRequest
Запрос на редактирование пользователей канала

У пользователей каналов могут быть различные полномочия. В зависимости от них пользователи могут совершать различные действия в каналах.

* Подписчики могут читать сообщения
* Администраторы могут отправлять сообщения, банить подписчиков
* Создатель может назначать администраторов, удалить канал

| Название | Тип значения | Описание |
| -| -| -| 
| Users | ChannelUser[] | Список отредактированных пользователей канала |
| ChannelId | Int64 | Идентификатор канала |
| RequestType | RequestType | Тип запроса. Значение: 50 |

Ответ на запрос: [ChannelUsersResponse](#2112-channelusersresponse)

#### 2.6.5.6 GetChannelUsersRequest
Запрос на получение списка пользователей канала

| Название | Тип значения | Описание |
| -| -| -| 
| ChannelId | Int64 | Идентификатор канала |
| NavigationUserId | Int64 | Идентификатор пользователя для постраничной навигации |
| RequestType | RequestType | Тип запроса. Значение: 51 |

Ответ на запрос: [ChannelUsersResponse](#2112-channelusersresponse)

### 2.6.6 GetConversationsRequest
Запрос на получение списка всех чатов, каналов и диалогов пользователя. См [ConversationPreview](APIObjects.md#44-conversationpreview)

| Название | Тип значения | Описание |
| -| -| -| 
| NavigationConversationId | Int64 | Идентификатор беседы для постраничной навигации |
| NavigationConversationType | ConversationType | Тип беседы для постраничной навигации |
| RequestType | RequestType | Тип запроса. Значение: 12 |

Ответ на запрос: [ConversationsResponse](#2116-conversationsresponse)

### 2.6.7 DeleteConversationRequest
Запрос на удаление чата/канала/диалога 

| Название | Тип значения | Описание |
| -| -| -| 
| ConversationId | Int64 | Идентификатор беседы |
| ConversationType | ConversationType | Тип беседы |
| RequestType | RequestType | Тип запроса. Значение: 5 |

Ответ на запрос: [ResultResponse](#2110-resultresponse)

## 2.7 Пользователи
### 2.7.1 GetUsersRequest
Запрос на получение информации о пользователях

| Название | Тип значения | Описание |
| -| -| -| 
| UsersId | Int64[] | Идентификаторы пользователей |
| IncludeContact | Boolean | Если true, объекты `User` придут с заполенным полем `User.Contact`  |
| RequestType | RequestType | Тип запроса. Значение: 28 |

Ответ на запрос: [UsersResponse](#21124-usersresponse)

### 2.7.2 ChangeEmailOrPhoneRequest
Запрос на смену email или номера телефона пользователя

| Название | Тип значения | Описание |
| -| -| -| 
| Value | String | Номер телефона или email |
| VCode | Int16 | Код верификации |
| RequestType | RequestType | Тип запроса. Значение: 77 |

Ответ на запрос: [UsersResponse](#21124-usersresponse)

## 2.8 Поиск

### 2.8.1 SearchRequest
Запрос на поиск пользователей/чатов/каналов

| Название | Тип значения | Описание |
| -| -| -| 
| SearchTypes | SearchType[] | Список типов сущностей, которые должны быть включены в результаты поиска |
| SearchQuery | String | Поисковый запрос |
| NavigationId | Int64 | Идентификатор сущности для постраничной навигации |
| Direction | Boolean | Направление навигации |
| RequestType | RequestType | Тип запроса. Значение: 46 |

Ответ на запрос: [SearchResponse](#21119-searchresponse)


### 2.8.2 SearchType

| Название | Значение | Описание     |
|----------|----------|--------------|
| Users    | 0        | Пользователи |
| Chats    | 1        | Чаты         |
| Channels | 2        | Каналы       |

## 2.9 Контакты и группы контактов
### 2.9.1 CreateOrEditContactRequest
Запрос на создание или редактирование контакта. См. [Contact](APIObjects.md#411-contact)


| Название | Тип значения | Описание |
| -| -| -| 
| Contact | Contact | Контакт |
| RequestType | RequestType | Тип запроса. Значение: 62 |

Ответ на запрос: [ContactResponse](#2115-contactsresponse)

### 2.9.2 GetContactsRequest
Запрос на получение списка контактов

| Название | Тип значения | Описание |
| -| -| -| 
| RequestType | RequestType | Тип запроса. Значение: 64 |

Ответ на запрос: [ContactResponse](#2115-contactsresponse)

### 2.9.3 CreateOrEditGroupRequest
Запрос на создание или редактирование группы контактов. См. [Group](APIObjects.md#410-group)

| Название | Тип значения | Описание |
| -| -| -| 
| Group | Group | Группа контактов |
| RequestType | RequestType | Тип запроса. Значение: 56 |

Ответ на запрос: [GroupsResponse](#21111-groupsresponse)

### 2.9.4 GetGroupsRequest
Запрос на получение списка групп

| Название | Тип значения | Описание |
| -| -| -| 
| RequestType | RequestType | Тип запроса. Значение: 61 |

Ответ на запрос: [GroupsResponse](#21111-groupsresponse)

### 2.9.5 GetGroupContactsRequest
Запрос на получение контактов группы

| Название | Тип значения | Описание |
| -| -| -| 
| GroupId | UUID | Идентификатор группы |
| RequestType | RequestType | Тип запроса. Значение: 60 |

Ответ на запрос: [ContactsResponse](#2115-contactsresponse)

### 2.9.6 DeleteContactsRequest
Запрос на удаление контактов

| Название | Тип значения | Описание |
| -| -| -| 
| ContactsId | UUID[] | Идентификаторы контактов |
| RequestType | RequestType | Тип запроса. Значение: 63 |

Ответ на запрос: [ResultResponse](#2110-resultresponse)

### 2.9.7 DeleteGroupsRequest
Запрос на удаление групп контактов

| Название | Тип значения | Описание |
| -| -| -| 
| GroupsId | UUID[] | Идентификаторы групп |
| RequestType | RequestType | Тип запроса. Значение: 57 |

Ответ на запрос: [ResultResponse](#2110-resultresponse)

## 2.10 Профиль
### 2.10.1 EditUserRequest
Запрос на редактирование информации и пользователе. См. [EditUser](APIObjects.md#411-edituser)

| Название | Тип значения | Описание |
| -| -| -| 
| User | EditUser | Отредактированный пользователь |
| RequestType | RequestType | Тип запроса. Значение: 10 |

Ответ на запрос: [UsersResponse](#21124-usersresponse)

### 2.10.2 EditFavoritesRequest
Запрос на создание/редактирование списка избранного пользователя. Для создания/редактирования нужно сформировать полный список объектов [Favorites](APIObjects.md#412-favorites)

| Название | Тип значения | Описание |
| -| -| -| 
| Favorites | Favorites[] | Избранное |
| RequestType | RequestType | Тип запроса. Значение: 69 |

Ответ на запрос: [FavoritesResponse](#2118-favoritesresponse)

### 2.10.3 Сессии пользователя
#### 2.10.3.1 GetSessionsRequest
Запрос на получение списка открытых сессий пользователя. См [Session](APIObjects.md#414-session)

| Название | Тип значения | Описание |
| -| -| -| 
| RequestId | Int64 | Идентификатор запроса |
| RequestType | RequestType | Тип запроса. Значение: 71 |

Ответ на запрос: [SessionsResponse](#21121-sessionsresponse)

#### 2.10.3.2 LogoutRequest
Запрос на закрытие сессий, используется для выхода из аккаунта на текущем или на других устройствах

| Название | Тип значения | Описание |
| -| -| -| 
| AccessToken | String | Access-токен |
| TokensIds | Int64[] | Идентификаторы токенов для закрытия определенных сессий; если NULL - закрывается текущая сессия |
| RequestType | RequestType | Тип запроса. Значение: 34 |

Ответ на запрос: [ResultResponse](#2110-resultresponse)

### 2.10.4 Перенос аккаунта на другой хаб
API предусматривает возможность полного перемещения данных пользователя на другой хаб. Для этого необходимо:
* Отправить на текущий хаб запрос `ChangeNodeRequest` для получения идентификатора операции переноса `OperationId`
* [Отправить GET-запрос](#21042-запуск-операции-перемещения-данных-пользователя) на хаб, на который необходимо переместиться

#### 2.10.4.1 ChangeNodeRequest
Запрос на создание операции перемещения пользователя на другой хаб

| Название | Тип значения | Описание |
| -| -| -| 
| NodeId | Int64 | Идентификатор хаба, на которую требуется переместить данные пользователя |
| RequestType | RequestType | Тип запроса. Значение: 200 |

Ответ на запрос: [OperationIdResponse](#21115-operationidresponse)

#### 2.10.4.2 Запуск операции перемещения данных пользователя
Для запуска операции перемещения данных пользователя необходимо отправить GET-запрос на хаб, на который требуется перенос данных

``` http
GET /UserMigration/StartDownloadingUserData? HTTP/1.1
Host: адрес хаба
operationId: value
nodeId: идентификатор хаба с который небходимо осуществить перемещение
```
После успешного переноса данных на устройства пользователя придет уведомление [UserNodeChangedNotice](#21211-usernodechangednotice).

## 2.11 Ответы на запросы

Все запросы наследуются от [Response](BasicConcepts.md#122-response)
### 2.11.0 ResultResponse
| Название | Тип значения | Описание |
| -| -| -| 
| Message | String | Описание ошибки, если она произошла |
| ResponseType | ResponseType | Тип ответа. Значение: 10 |

### 2.11.1 ChannelsResponse
| Название | Тип значения | Описание |
| -| -| -| 
| Channels | Channel[] | Список каналов |
| ResponseType | ResponseType | Тип ответа. Значение: 20 |

### 2.11.2 ChannelUsersResponse
| Название | Тип значения | Описание |
| -| -| -| 
| Administration | ChannelUser[] | Список администраторов канала |
| Subscribers | ChannelUser[] | Список подписчиков канала |
| BlockedUsers | ChannelUser[] | Список заблокированных подписчиков канала |
| ResponseType | ResponseType | Тип ответа. Значение: 21 |

### 2.11.3 ChatsResponse
| Название | Тип значения | Описание |
| -| -| -| 
| Chats | Chat[] | Список чатов |
| ResponseType | ResponseType | Тип ответа. Значение: 2 |

### 2.11.4 ChatUsersResponse
| Название | Тип значения | Описание |
| -| -| -| 
| ChatUsers | ChatUser[] | Список пользователей чата |
| ResponseType | ResponseType | Тип ответа. Значение 16 |

### 2.11.5 ContactsResponse
| Название | Тип значения | Описание |
| -| -| -| 
| Contacts | Contact[] | Список контактов |
| ResponseType | ResponseType | Тип ответа. Значение: 27 |
### 2.11.6 ConversationsResponse
| Название | Тип значения | Описание |
| -| -| -| 
| Conversations | ConversationPreview[] | Список чатов/диалогов/каналов пользователя |
| ResponseType | ResponseType | Тип ответа. Значение: 15 |

### 2.11.7 EncryptedKeyResponse
| Название | Тип значения | Описание |
| -| -| -| 
| EncryptedData | String | Зашифрованный ключ в формате Base64 |
| ResponseType | ResponseType | Тип ответа. Значение: 22 |

### 2.11.8 FavoritesResponse
| Название | Тип значения | Описание |
| -| -| -| 
| Favorites | Favorites[] | Список избранного |
| ResponseType | ResponseType | Тип ответа. Значение: 28 |

### 2.11.9 FileResponse
| Название | Тип значения | Описание |
| -| -| -| 
| File | FileInfo | Объект, содержащий информацию о файле |
| FileAccessToken | String | Токен для идентификации пользователя при загрузке файла на хаб |
| ResponseType | ResponseType | Тип ответа. Значение: 7 |

### 2.11.10 FilesResponse
| Название | Тип значения | Описание |
| -| -| -| 
| Files | FileInfo[] | Список файлов |
| ResponseType | ResponseType | Тип ответа. Значение: 8 |

### 2.11.11 GroupsResponse
| Название | Тип значения | Описание |
| -| -| -| 
| Groups | Group[] | Список групп контактов |
| ResponseType | ResponseType | Тип ответа. Значение: 26 |

### 2.11.12 KeysResponse
| Название | Тип значения | Описание |
| -| -| -| 
| Keys | Key[] | Список публичных ключей пользователя |
| ResponseType | ResponseType | Тип ответа. Значение: 18 |

### 2.11.13 MessagesResponse
| Название | Тип значения | Описание |
| -| -| -| 
| Messages | Message[] | Список сообщений |
| ResponseType | ResponseType | Тип ответа. Значение: 9 |

### 2.11.14 NodesResponse
| Название | Тип значения | Описание |
| -| -| -| 
| Nodes | Node[] | Список хабов |
| ResponseType | ResponseType | Тип ответа. Значение: 1 |

### 2.11.15 OperationIdResponse
| Название | Тип значения | Описание |
| -| -| -| 
| OperationId | String | Идентификатор операции переноса данных пользователя на другой хаб |
| ResponseType | ResponseType | Тип ответа. Значение: 24 |

### 2.11.16 PollResponse
| Название | Тип значения | Описание |
| -| -| -| 
| Poll | Poll | Объект, содержащий информацию об опросе |
| ResponseType | ResponseType | Тип ответа. Значение: 25 |

### 2.11.17 PollResultsResponse
| Название | Тип значения | Описание |
| -| -| -| 
| PollResults | VoteInfo[] | Список объектов, содержащих информацию о результатах опроса |
| ResponseType | ResponseType | Тип ответа. Значение: 31 |

### 2.11.18 QRCodeResponse
| Название | Тип значения | Описание |
| -| -| -| 
| QR | QRCodeContent | Объект, содержащий информацию, хранящуюся в QR-коде |
| ResponseType | ResponseType | Тип ответа. Значение: 30 |

### 2.11.19 SearchResponse
| Название | Тип значения | Описание |
| -| -| -| 
| Users | User[] | Список пользователей |
| Chats | Chat[] | Список чатов |
| Channels | Channel[] | Список каналов |
| ResponseType | ResponseType | Тип ответа. Значение: 23 |

### 2.11.20 SequenceResponse
| Название | Тип значения | Описание |
| -| -| -| 
| Sequence | String | Случайная последовательность в формате Base64 |
| ResponseType | ResponseType | Тип ответа. Значение: 19 |

### 2.11.21 SessionsResponse
| Название | Тип значения | Описание |
| -| -| -| 
| Sessions | Session[] | Список открытых сессий пользователя|
| ResponseType | ResponseType | Тип ответа. Значение: 29 |

### 2.11.22 TokensResponse
| Название | Тип значения | Описание |
| -| -| -| 
| User | User | Информация о пользователе |
| Token | Token | Токены пользователя |
| FileAccessToken | String | Токен для идентификации пользователя при загрузке файлов на хаб |
| ResponseType | ResponseType | Тип ответа. Значение: 6 |

### 2.11.23 UpdatedMessagesResponse
| Название | Тип значения | Описание |
| -| -| -| 
| Start | Int64 | Время изменения/удаления первого сообщения из списков Deleted и Updated |
| End | Int64 | Время изменения/удаления последнего сообщения из списков Deleted и Updated |
| EndConversationId | Int64 | Идентификатор беседы, в которой было изменено/удалено последнее сообщение |
| EndConversationType | ConversationType | Тип беседы, в которой было изменено/удалено последнее сообщение |
| EndMessageId | UUID | Идентификатор последнего измененного/удаленного сообщения |
| Deleted | MessageInfo[] | Список объектов, содержащих информацию об удаленных сообщениях |
| Updated | Message[] | Список обновленных сообщений |
| ResponseType | ResponseType | Тип ответа. Значение: 17 |

### 2.11.24 UsersResponse
| Название | Тип значения | Описание |
| -| -| -| 
| Users | User[] | Список пользователей |
| ResponseType | ResponseType | Тип ответа. Значение: 0 |

## 2.12 Уведомления от хаба
Все уведомления наследуются от [Notice](BasicConcepts.md#123-notice)

### 2.12.1 ChannelNotice
Данное уведомление приходит пользователям канала при его создании/редактировании

| Название | Тип значения | Описание |
| -| -| -| 
| Channel | Channel | Канал |
| Code | NoticeCode | Код уведомления. Значение: 10 |

### 2.12.2 ChatUsersChangedNotice
Данное уведомление приходят пользователям чата при измении/добавлении пользователей чата

| Название | Тип значения | Описание |
| -| -| -| 
| ChatUsers | ChatUser[] | Список пользователей чата |
| Code | NoticeCode | Код уведомления. Значение: 8 |

### 2.12.3 ConversationActionNotice
Данное уведомление приходит, когда какой-либо пользователь из беседы отправляет специальный запрос `ConversationActionRequest`

| Название | Тип значения | Описание |
| -| -| -| 
| Action | ConversationAction | Тип действия пользователя |
| UserId | Int64 | Идентификатор пользователя |
| ConversationId | Int64 | Идентификатор беседы |
| ConversationType | ConversationType | Тип беседы |
| Code | NoticeCode | Код уведомления. Значение: 14 |

### 2.12.4 EditChatNotice
Данное уведомление приходит при редактировании информации о чате

| Название | Тип значения | Описание |
| -| -| -| 
| Chat | Chat | Чат |
| Code | NoticeCode | Код уведомления. Значение: 7 |

### 2.12.5 EncryptedKeyNotice
Данное уведомление приходит в результате запроса пользователя на получение приватных ключей с других устройств

| Название | Тип значения | Описание |
| -| -| -| 
| DeviceOSName | String | Название ОС устройства|
| AppName | String | Название приложения |
| DeviceName | String | Название устройства |
| EncryptedSymmetricKey | String | Зашифрованный симметричный ключ в формате Base64 |
| EncryptedKeys | String | Зашифрованные симметричным ключом ключи |
| PublicKey | String | Публичный ключ подписи |
| Code | NoticeCode | Код уведомления. Значение: 12 |

### 2.12.6 MessagesReadedNotice
Данное уведомление приходит отправителю сообщений, когда другой пользователь помечает сообщения отправителя прочтенными с помощью запроса `MessagesReadRequest`

| Название | Тип значения | Описание |
| -| -| -| 
| MessagesId | UUID[] | Идентификаторы сообщений |
| ConversationId | Int64 | Идентификатор беседы |
| ConversationType | ConversationType | Тип беседы |
| Code | NoticeCode | Код уведомления. Значение: 5 |

### 2.12.7 MessagesUpdatedNotice
Данное уведомление приходит пользователям при удалении/редактировании сообщений

| Название | Тип значения | Описание |
| -| -| -| 
| Deleted | MessageInfo | Объекты, содержащие информацию об удаленных сообщения |
| Updated | Message[] | Отредактированные сообщения |
| Code | NoticeCode | Код уведомления. Значение: 9 |

### 2.12.8 NewChatNotice
Данное уведомление отправляется пользователям чата при его создании

| Название | Тип значения | Описание |
| -| -| -| 
| Chat | Chat | Чат |
| Code | NoticeCode | Код уведомленияю Значение: 3 |

### 2.12.9 NewMessageNotice
Данное уведомление приходит при получении нового сообщения

| Название | Тип значения | Описание |
| -| -| -| 
| Message | Message | Сообщение |
| Call | Boolean | Содержит ли пришедшее сообщение обращение к пользователю |
| Code | NoticeCode | Код уведомления. Значение: 0 |

### 2.12.10 NewSessionNotice
Данное уведомление приходит, когда пользователь авторизуется на другом устройстве

| Название | Тип значения | Описание |
| -| -| -| 
| Session | Session | Информация о новой сессии |
| Code | NoticeCode | Код уведомления. Значение: 13 |

### 2.12.11 UserNodeChangedNotice
Данное уведомление приходит на устройства пользователя, если данные пользователя были перемещены на другой хаб 

| Название | Тип значения | Описание |
| -| -| -| 
| NewNodeId | Int64 | Идентификатор нового хаба |
| UserId | Int64 | Идентификатор пользователя |
| Code | NoticeCode | Код уведомления. Значение: 11 |

### 2.12.12 UsersAddedNotice
Данное уведомление приходит на устройства пользователя, когда в чат добавляются новые пользователи

| Название | Тип значения | Описание |
| -| -| -| 
| NewUsers | ChatUser[] | Пользователи чата |
| Chat | Chat | Чат |
| Code | NoticeCode | Код уведомления. Значение: 4 |


## 2.13 Работа с файлами
### 2.13.1 Загрузка файла на хаб
Для загрузки файла на хаб необходимо отправить POST-запрос:
``` http
POST https://Node.Domains[0]:Node.ClientsPort/api/Files?isDocument=true HTTP/1.1
FileAccessToken: Token.FileAccessToken (см. TokensResponse)
Connection: Keep-Alive
Content-Type: multipart/form-data; boundary=---------------------
8d6b6ae18202cb7
Content-Length: 114105
Host: Node.Domains[0]:Node.ClientsPort
-----------------------8d6b6ae18202cb7
Content-Disposition: form-data; name="file";
filename="file"
Content-Type: application/octet-stream
*Содержимое файла*
```
Запрос имеет query-параметр `isDocument`. Если его значение `TRUE`, загруженные изображения не будут сжиматься.

Пример ответа с информацией о файле:
``` http
HTTP/1.1 200 OK
Date: Mon, 01 Apr 2019 07:26:58 GMT
Content-Type: application/json
Server: Kestrel
Content-Length: 371
{"File":{"FileId":"DZSABkoclRMMSqihQ9BLsgdoQPoxWS6D1u0IdVMyr76XF2d1r
eypr2FepDlBM4oc","NumericId":20,"Filename":"file","Ha
sh":"tR0WHi0u7mMLe3ixiW5xaQ==","Uploaded":1554103618,"UploaderId":12
7,"NodeId":1,"Size":113902},"FileAccessToken":"jvQYkg7kdourxz2jFpZQJ
NXoYjQaa98aKKbjKjqWPw6d5S5iBpCAuqmf5612ffnV","RequestId":0,"Response
Type":7,"ErrorCode":0,"Type":1}
```
### 2.13.2 Скачивание файла с хаба
Для скачивания файла с хаба необходимо отправить GET-запрос:

``` http
GET /api/Files/File-Id? HTTP/1.1
Host: Node.Domains[0]:Node.ClientsPort
```
### 2.13.3 Удаление файла
Для удаления файла нужно отправить запрос следующего вида:

| Название | Тип значения | Описание |
| -| -| -| 
| FilesId | String[] | Идентификаторы файлов |
| RequestType | RequestType | Тип запроса. Значение: 6 |

Ответ на запрос: [ResultResponse](#2110-resultresponse)

## 2.14 Голосования
### 2.14.1 Создание голосования
Для создания голосования необходимо отправить сообщение со [специальным вложением](APIObjects.md#49-poll). Такое сообщение возможно отправить только в чат или канал. 
### 2.14.2 Отправка голоса
Для голосования в опросе необходимо отправить запрос `PollingRequest`

| Название | Тип значения | Описание |
| -| -| -| 
| PollId | UUID | Идентификатор опроса |
| ConversationId | Int64 | Идентификатор беседы |
| ConversationType | ConversationType | Тип беседы |
| Options | PollVote[] | Варианты ответа |
| RequestType | RequestType | Тип запроса. Значение: 53 |

Ответ на запрос: [PollResponse](#21116-pollresponse)

### 2.14.3 Получение результатов голосования
Для получения результатов голосования необходимо отправить запрос `GetPollVotedUsersRequest`

| Название | Тип значения | Описание |
| -| -| -| 
| PollId | UUID | Идентификатор опроса |
| OptionId | Int8 | Идентификатор варианта ответа |
| ConversationType | ConversationType | Тип беседы |
| ConversationId | Int64 | Идентификатор беседы |
| NavigationUserId | Int64 | Идентификатор пользователя для постраничной навигации |
| RequestType | RequestType | Тип запроса. Значение: 54 |

Ответ на запрос: [PollResultsResponse](#21117-pollresultsresponse)

## 2.15 Синхронизация приватных ключей между устройствами пользователя
Чтобы пользователям было удобно работать с шифрованием, API предусматривает возможность безопасного обмена приватными ключами между устройствами пользователей.
### 2.15.1 Отправка запроса на хаб для получения ключей
Запрос `GetDevicesPrivateKeysRequest`

| Название | Тип значения | Описание |
| -| -| -| 
| PublicKey | String | Публичный ключ шифрования в формате Base64 |
| SignKeyId | Int64 | Идентификатор ключа подписи |
| Sign | String | Подпись публичного ключа в формате Base64 |
| RequestType | RequestType | Тип запроса. Значение: 70 |
 Ответ на запрос: [ResultResponse](#2110-resultresponse)

### 2.15.2 Обработка запроса от хаба на получение приватных ключей и отправка ответа
После отправки запроса `GetDevicesPrivateKeysRequest` на другие активные устройства пользователя придет запрос от хаба на получение приватных ключей `GetKeysClientRequest`:

| Название | Тип значения | Описание |
| -| -| -| 
| PublicKey | String | Публичный ключ шифрования в формате Base64 |
| SignKeyId | Int64 | Идентификатор ключа подписи |
| Sign | String | Подпись публичного ключа в формате Base64 |
| RequestType | ClientRequestType | Тип запроса. Значение: 1 |

Клиент должен проверить подпись, и, если подпись верна, отправить ответ хабу `EncryptedDataClientResponse`:

| Название | Тип значения | Описание |
|-|-|-|
| EncryptedSymmetricKey | String | Зашифрованный симметричный ключ на публичном ключе пользователя из запроса `GetKeysClientRequest`|
| EncryptedData | String | Массив объектов [AsymmetricKey](APIObjects.md#419-asymmetrickey), переведенный в формат JSON и затем зашифрованный симметричным ключом |
| PublicKey | String | Публичный ключ подписи |
| ResponseType | ClientResponseType | Тип ответа. Значение: 1 |

После получения ответа, хаб отправит уведомление [EncryptedKeyNotice](#2125-encryptedkeynotice) на устройство пользователя, запросившее ключи.