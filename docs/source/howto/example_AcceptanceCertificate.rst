﻿Как отправить и получить акт о выполнении работ/оказании услуг в рекомендованном ФНС формате
============================================================================================

Рассмотрим последовательность действий к функциям интеграторского интерфейса Диадока, которые требуется совершить при отправке акта о выполнении работ/оказании услуг.

#. Исполнитель формирует файл титула исполнителя, подписывает и направляет заказчику.

#. Заказчик получает товарную накладную, подписанную и отправленную исполнителем;

#. Заказчик формирует файл титула заказчика, подписывает своей ЭП и отправляет в адрес исполнителя.


.. note:: Более подробно о порядке обмена электронными актами о выполнении работ/оказании услуг между компаниями можно почитать на `сайте <http://www.diadoc.ru/docs/others/acts>`__

Формирование файла титула исполнителя для акта о выполнении работ/оказании услуг
--------------------------------------------------------------------------------

Если на стороне интеграционного решения не предусмотрено функциональности для формирования XML-документов, соответствущих утвержденным форматам, то исполнитель может сгенерировать файл титула, используя команду :doc:`../http/GenerateAcceptanceCertificateXmlForSeller`.
	   
В теле запроса должны содержаться данные для изготовления титула исполнителя для акта о выполнении работ/оказании услуг в XML-формате в виде сериализованной структуры :doc:`AcceptanceCertificateSellerTitleInfo <../proto/AcceptanceCertificateInfo>`.
	   
HTTP-запрос для генерации файла титула исполнителя акта о выполнении работ/оказании услуг выглядит следующим образом:

::

    POST /GenerateAcceptanceCertificateXmlForSeller HTTP/1.1
    Host: diadoc-api.kontur.ru
    Authorization: DiadocAuth ddauth_api_client_id=testClient-8ee1638deae84c86b8e2069955c2825a
    Content-Length: 1252
    Connection: Keep-Alive

    <Сериализованная структура AcceptanceCertificateSellerTitleInfo>

В теле ответа содержится XML-файл титула исполнителя, построенный на основании данных из запроса.

Успешный ответ сервера выглядит так:
::

    HTTP/1.1 200 OK
    Content-Length: 598

    <XML-файл титула исполнителя>

Файл генерируется в соответствии с `XML-схемой <https://diadoc.kontur.ru/sdk/xsd/DP_ZAKTPRM_1_990_00_05_01_02.xsd>`__, которой должны удовлетворять XML-акты, согласно приказу ФНС.


Имя файла титула исполнителя для акта о выполнении работ/оказании услуг возвращается в стандартном HTTP-заголовке ``Content-Disposition``.

Отправка файла титула исполнителя для акта о выполнении работ/оказании услуг
----------------------------------------------------------------------------

После того, как у вас есть XML-файл титула исполнителя, его нужно отправить с помощью команды :doc:`../http/PostMessage`. 

Для этого нужно подготовить структуру :doc:`../proto/MessageToPost` следующим образом:

-  в значение атрибута *FromBoxId* указываем идентификатор ящика отправителя;

-  в значение атрибута *ToBoxId* указываем идентификатор ящика получателя;

-  для передачи XML-файла титула исполнителя акта о выполнении работ/оказании услуг нужно использовать атрибут *XmlAcceptanceCertificateSellerTitles*, описываемый структурой *XmlDocumentAttachment*:

	-  внутри структуры *XmlDocumentAttachment* находится вложенная структура *SignedContent*;
	
	-  сам XML-файл нужно передать в атрибут *Content*, подпись исполнителя в атрибут *Signature*.
	   
Описание структур, используемых при отправке акта о выполнении работ/оказании услуг:

.. code-block:: protobuf

    message MessageToPost {
        required string FromBoxId = 1;
        optional string ToBoxId = 2;
        repeated XmlDocumentAttachment XmlAcceptanceCertificateSellerTitles = 3;
    }

    message XmlDocumentAttachment {
        required SignedContent SignedContent = 1;
    }

    message SignedContent {
        optional bytes Content = 1;
        optional bytes Signature = 2;
    }

После отправки в теле ответа будет содержаться отправленное сообщение, сериализованное в протобуфер :doc:`../proto/Message`.

Все дальнейшие действия происходят на стороне покупателя.

Поиск акта о выполнении работ/оказании услуг
--------------------------------------------

Сначала покупателю необходимо найти все входящие акты о выполнении работ/оказании услуг, которые требуется обработать. Для этого нужно воспользоваться методом :doc:`../http/GetDocuments`:

  -  в значении параметра *boxId* указываем идентификатор ящика, в котором следует выполнить поиск входящих документов;

  -  в параметр *filterCategory* указываем статус и тип документа: ``XmlAcceptanceCertificate.InboundNotFinished``.

Пример запроса на получение акта о выполнении работ/оказании услуг выглядит следующим образом:

::

    GET /V3/GetDocuments?filterCategory=XmlAcceptanceCertificate.InboundNotFinished&boxId=db32772b-9256-49a8-a133-fda593fda38a HTTP/1.1
    Host: diadoc-api.kontur.ru
    Accept: application/json
    Content-Type: application/json charset=utf-8
    Authorization: DiadocAuth ddauth_api_client_id=testClient-87e1638deae84c86b8e2069955c2825a0987

В теле ответа вернется список документов в виде структуры *DocumentList* с вложенной структурой *Document*. Для каждого из этих документов запоминаем: *MessageId*, *EntityId*.

Получение акта о выполнении работ/оказании услуг
------------------------------------------------

Теперь необходимо получить найденный акт :doc:`XmlAcceptanceCertificate <../proto/Entity message>`.

Чтобы получить акт о выполнении работ/оказании услуг нужно вызвать метод :doc:`../http/GetMessage` и указать нужные GET-параметры ``boxId``, ``messageId``, ``entityId``.

``BoxId`` - это идентификатор ящика получателя, ``messageId`` - идентификатор полученного сообщения с актом о выполнении работ/оказании услуг, ``entityId`` - идентификатор акта. Их можно взять из структуры :doc:`../proto/Message`.

::

    GET /V3/GetMessage?messageId=bbcedb0d-ce34-4e0d-b321-3f600c920935&entityId=30cf2c07-7297-4d48-bc6f-ca7a80e2cf95&boxId=db32772b-9256-49a8-a133-fda593fda38a HTTP/1.1
    Host: diadoc-api.kontur.ru
    Accept: application/json
    Content-Type: application/json charset=utf-8
    Authorization: DiadocAuth ddauth_api_client_id=testClient-87e1638deae84c86b8e2069955c2825a0987

Пример структуры акта о выполнении работ/оказании услуг :doc:`XmlAcceptanceCertificate <../proto/Entity message>` в теле ответа:

.. code-block:: json

   {
       "EntityType": "Attachment",
       "EntityId": "654ac483-0dd4-4085-b70f-565c8b754e10",
       "Content": "lores ipsum",
       "AttachmentType": "XmlAcceptanceCertificate",
       "FileName": "DP_ZAKTPRM_2BM-7750370234-4012052808304878702630000000000_2BM_20150927_324c290e-f049-4906-baac-1ddcd7f3c2ff.xml",
       "NeedRecipientSignature": true,
       "SignerBoxId": "",
       "NotDeliveredEventId": "",
       "RawCreationDate": 635789700936777240,
       "SignerDepartmentId": "",
       "NeedReceipt": false,
       "IsApprovementSignature": false,
       "IsEncryptedContent": false
   }

.. _create_receipt:

Формирование файла титула заказчика для акта о выполнении работ/оказании услуг
------------------------------------------------------------------------------

Файл титула заказчика можно сформировать как на стороне интеграционного решения, так и используя команду :doc:`../http/GenerateAcceptanceCertificateXmlForBuyer`. Для этого надо передать следующие параметры: 

- ``boxId`` - идентификатор ящика получателя;

- ``sellerTitleMessageId`` - идентификатор сообщения, содержащего соответствующий титул исполнителя;

- ``sellerTitleAttachmentId`` - идентификатор сущности, представляющей титул исполнителя, для которого требуется изготовить титул заказчика.

Эти идентификаторы соответствуют идентификаторам из параметров ``boxId``, ``messageId``, ``entityId`` для метода :doc:`../http/GetMessage`.
	   
В теле запроса должны содержаться данные для изготовления титула заказчика для акта о выполнении работ/оказании услуг в XML-формате в виде сериализованной структуры :doc:`AcceptanceCertificateBuyerTitleInfo <../proto/Torg12Info>`.
	   
HTTP-запрос для генерации файла титула заказчика акта о выполнении работ/оказании услуг выглядит следующим образом:

::

    POST /GenerateAcceptanceCertificateXmlForBuyer?sellerTitleMessageId=bbcedb0d-ce34-4e0d-b321-3f600c920935&sellerTitleAttachmentId=30cf2c07-7297-4d48-bc6f-ca7a80e2cf95&boxId=db32772b-9256-49a8-a133-fda593fda38a HTTP/1.1
    Host: diadoc-api.kontur.ru
    Authorization: DiadocAuth ddauth_api_client_id=testClient-8ee1638deae84c86b8e2069955c2825a
    Content-Length: 1252
    Connection: Keep-Alive

    <Сериализованная структура AcceptanceCertificateBuyerTitleInfo>

В теле ответа содержится XML-файл титула заказчика, построенный на основании XML-файла титула исполнителя и данных из запроса.

Успешный ответ сервера выглядит так:
::

    HTTP/1.1 200 OK
    Content-Length: 598

    <XML-файл титула заказчика>

Файл генерируется в соответствии с `XML-схемой <https://diadoc.kontur.ru/sdk/xsd/DP_ZAKTPRM_1_990_00_05_01_02.xsd>`__, которой должны удовлетворять XML-акты, согласно приказу ФНС.


Имя файла титула заказчика для акта о выполнении работ/оказании услуг возвращается в стандартном HTTP-заголовке ``Content-Disposition``.

Отправка файла титула заказчика для акта о выполнении работ/оказании услуг
--------------------------------------------------------------------------
После того, как у вас есть XML-файл титула заказчика, его нужно отправить с помощью команды :doc:`../http/PostMessagePatch`. 

Для этого нужно подготовить структуру :doc:`../proto/MessagePatchToPost` следующим образом:

-  в значение атрибута *BoxId* указываем идентификатор ящика, в котором находится исходное сообщение;

-  в значение атрибута *MessageId* указываем идентификатор сообщения, к которому относится отправляемый патч;

-  для передачи XML-файла титула исполнителя акта о выполнении работ/оказании услуг нужно использовать атрибут *XmlAcceptanceCertificateBuyerTitles*, описываемый структурой *ReceiptAttachment*:

    -  ParentEntityId - идентификатор документа, к которому относится титул заказчика; это идентификатор соответствующей сущности из родительского сообщения (поле EntityId в структуре :doc:`Entity <../proto/Entity message>`.);

	-  внутри структуры *ReceiptAttachment* находится вложенная структура *SignedContent*;
	
	-  сам XML-файл нужно передать в атрибут *Content*, подпись исполнителя в атрибут *Signature*.
	   
Описание структур, используемых при отправке акта о выполнении работ/оказании услуг:

.. code-block:: protobuf

    message MessagePatchToPost {
        required string BoxId = 1;
        optional string MessageId = 2;
        repeated ReceiptAttachment XmlAcceptanceCertificateBuyerTitles = 7;
    }

    message ReceiptAttachment {
		required string ParentEntityId = 1;
        required SignedContent SignedContent = 1;
    }

    message SignedContent {
        optional bytes Content = 1;
        optional bytes Signature = 2;
    }

После отправки в теле ответа будет содержаться отправленное дополнение, сериализованное в протобуфер :doc:`../proto/MessagePatch`.

SDK
---

Пример кода на C# для отправки файла титула исполнителя для акта о выполнении работ/оказании услуг:

.. code-block:: csharp

	// формирование файла титула исполнителя
	private GeneratedFile GenerateAcceptanceCertificateSellerTitle()
	{
		var content = new AcceptanceCertificateSellerTitleInfo()
		{
			// заполняем согласно структуре AcceptanceCertificateSellerTitleInfo
		};
		return api.GenerateAcceptanceCertificateXmlForSeller(authToken, content);
	}
		
	// отправка файла титула исполнителя
	private void SendAcceptanceCertificateSellerTitle()
	{
		var fileToSend = GenerateAcceptanceCertificateSellerTitle();

		var messageAttachment = new XmlDocumentAttachment
		{
			SignedContent = new SignedContent //файл подписи
			{
				Content = fileToSend.Content,
				Signature = new byte[0] //подпись исполнителя
			}
		};

		var messageToPost = new MessageToPost
		{
			FromBoxId = "идентификатор ящика отправителя",
			ToBoxId = "идентификатор ящика получателя",
			XmlAcceptanceCertificateSellerTitles = { messageAttachment }
		};

		api.PostMessage(authToken, messageToPost); //см. "Как авторизоваться в системе"
	}

	
Пример кода на C# для получения файла титула исполнителя для акта о выполнении работ/оказании услуг и отправки файла титула заказчика:

.. code-block:: csharp

	//находим все неподписанные акты о выполнении работ/оказании услуг
	private Document SearchAcceptanceCertificateDocuments()
	{
		var boxId = "идентификатор ящика, в котором следует выполнить поиск входящих документов";
			
		//статус и тип документа
		var filterCategory = "XmlAcceptanceCertificate.InboundNotFinished"; 
		var counteragentBoxId = "идентификатор ящика контрагента";
			
		DocumentList documents = api.GetDocuments(authToken, boxId, filterCategory, counteragentBoxId);
			
		return documents.Documents.First();
	}
		
	//получаем нужный акт и отправляем к нему титул заказчика
	private void GetAcceptanceCertificateAndSendBuyerTitle()
	{
		var document = SearchAcceptanceCertificateDocuments();
		var boxId = "идентификатор ящика получателя";
			
		//получение акта о выполнении работ/оказании услуг
		var message = api.GetMessage(authToken, boxId, document.MessageId, document.EntityId); 
			
		//генерация файла титула заказчика
		var buyerInfo = GetBuyerInfo();
		
		//формирование файла титула заказчика
		var acceptanceCertificateXmlForBuyer = api.GenerateAcceptanceCertificateXmlForBuyer(authToken, buyerInfo, boxId, document.MessageId, document.EntityId);
	
		var messagePatchToPost = new MessagePatchToPost
		{
			BoxId = boxId,
			MessageId = document.MessageId,
			XmlAcceptanceCertificateBuyerTitles =
			{
				new ReceiptAttachment
				{
					ParentEntityId = document.EntityId,
					SignedContent = new SignedContent //файл подписи
					{
						Content = document.Content,
						Signature = new byte[0] //подпись заказчика
					}
				}
			}
		};

		api.PostMessagePatch(authToken, messagePatchToPost);
	}
	
	//генерация файла титула заказчика
	private AcceptanceCertificateBuyerTitleInfo GetBuyerInfo()
	{
		return new AcceptanceCertificateBuyerTitleInfo
		{
			//заполняется согласно структуре AcceptanceCertificateBuyerTitleInfo
		};
	}