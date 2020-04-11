# hellorpc
windows rpc 使用MIDL+RPC实现HelloWorld
见[csdn 博客说明](http://blog.csdn.net/xiaoqing_2014/article/details/79624866)


    Aus Sicht des Betriebssystems werden viele Komponentendienste integriert und einige Aufrufstandards für Entwickler bereitgestellt. In Windows ist die com-Komponente und der rpc-Aufruf am häufigsten. Das Hauptthema dieses Artikels bezieht sich auf die Entführung von RPCs. Ich glaube, dass Studenten, die IPC unabhängig geschrieben haben, nicht mit RPC vertraut sind. Die Effizienz von RPC bei der prozessübergreifenden Kommunikationsmethode hat offensichtliche Vorteile gegenüber anderen Methoden. Häufig verwendete Kommunikationsmethoden. Microsoft hat jedoch nicht viel Dokumentation zur Verwendung von rpc bereitgestellt, insbesondere zu seinen Kommunikationsstandards und Kommunikationsmethoden, wie z. B. der Auswahl von Kommunikationskanälen, Sockets und AlpcPort / Port-Protokollen oder synchron und asynchron Es gibt einige Schwellenwerte für die Anrufauswahl. In diesem Artikel wird jedoch nicht erläutert, wie Sie rpc für die Kommunikation zwischen Prozessen verwenden. Sie möchten jedoch einige Komponenten des Windows-Betriebssystems verstehen, die rpc verwenden, und es entführen.

0x01 Herkunft
        Bevor die Schlussfolgerung geschlossen wurde, untersuchte der Autor sie zunächst, da einige Anbieter auf Windows-Systemdienstebene ein starkes Interesse an der HIPS-Technologie hatten. Der größte Teil des Codes der Anwendungsschicht ist auch ein kleines Modul, das von einem Produkt umgekehrt ist. Der Autor hat keine anderen Versuche, nur dieses Thema zu studieren, um zu lernen und zu wachsen. Nach dem Studium dieses Moduls kam ich zu folgendem Schluss:

       1. HIPS für die Erstellung / Änderung von Diensten ist kein Abfangen der Registrierung auf Kernelebene, sondern eine geheime Operation, die in services.exe eingefügt wird.

       2. Dieses Modul der Anwendungsschicht missbraucht einige Funktionen von RPCRT4.dll, dh den Registrierungsprozess von RPC Server, und implementiert Blockierungsregeln synchron mit dem Kernelmodul.

       3. services.exe hat ein relativ hohes TOKEN-Token. Es ist praktisch, die Attributinformationen des Clients abzurufen.
Folgen Sie dann diesem Gedankengang und machen Sie selbst einige Versuche.
0x02 verwandte Struktur
       Wenn Sie ein Anfänger in diesem Bereich sind, wird empfohlen, zu RPCDemo zu springen, um zu lernen.
       Um Ihren eigenen RPC zu erstellen, benötigen Sie zunächst eine Beschreibung der IDL-Schnittstelle und generieren dann mit dem MIDL-Tool die entsprechenden Client- und Server-Stub-Dateien.

Server:
RPCRTAPI
RPC_STATUS
RPC_ENTRY
RpcServerRegisterIf (
    _In_ RPC_IF_HANDLE IfSpec,
    _In_opt_ UUID __RPC_FAR * MgrTypeUuid,
    _In_opt_ RPC_MGR_EPV __RPC_FAR * MgrEpv
    );

typedef void __RPC_FAR * RPC_IF_HANDLE;

Der Server muss registriert sein. Die wichtigste Struktur ist RPC_IF_HANDLE. Auf dem Server lautet diese Zeigerstruktur:

struct _RPC_SERVER_INTERFACE typedef
{
    unsigned int die Länge;
    RPC_SYNTAX_IDENTIFIER InterfaceID;
    RPC_SYNTAX_IDENTIFIER TransferSyntax;
    PRPC_DISPATCH_TABLE DispatchTable;
    unsigned int RpcProtseqEndpointCount;
    PRPC_PROTSEQ_ENDPOINT RpcProtseqEndpoint;
    RPC_MGR_EPV __RPC_FAR * DefaultManagerEpv;
    nichtig const * __RPC_FAR InterpreterInfo;
    unsigned int die Flags;
} RPC_SERVER_INTERFACE, __RPC_FAR * PRPC_SERVER_INTERFACE;

Dann ist es nicht die DispatchTable, die Aufmerksamkeit benötigt, sondern das InterpreterInfo-Mitglied, das tatsächlich:

struct _MIDL_SERVER_INFO_ typedef
    {
    PMIDL_STUB_DESC pStubDesc;
    const * SERVER_ROUTINE DispatchTable;
    PFORMAT_STRING ProcString;
    const unsigned Short * FmtStringOffset;
    const * STUB_THUNK ThunkTable;
    PRPC_SYNTAX_IDENTIFIER pTransferSyntax;
    ein ULONG_PTR nCount;
    PMIDL_SYNTAX_INFO pSyntaxInfo;
    } MIDL_SERVER_INFO, * PMIDL_SERVER_INFO;

In der DispatchTable hier kümmern wir uns. Dann können Sie für die Schlüsselprozessdienste.exe von Windows einige seiner internen Logik sehen:


140072320 ist eigentlich die RPC_SERVER_INTERFACE-Struktur. Wir können feststellen, dass InterpreterInfo bei 140073D10 liegt


Die Struktur von 140073D10 lautet MIDL_SERVER_INFO, und das zweite Mitglied ist die DispatchTable des Servers


Wenn Sie also so viele servicebezogene Funktionstabellen gesehen haben, können Sie die Erwartungen erfüllen, wenn Sie die wichtigsten Punkte in dieser Tabelle entführen


0x03 Reverse Push Realisierung
       Da RPCRT4.dll verschiedene Dienstregistrierungsschnittstellen in verschiedenen Versionen bereitstellt, gibt es einige Kompatibilitätsprobleme, sodass es nicht möglich ist, nur alle Funktionen zu missbrauchen, über die möglicherweise zugegriffen wird.

Hijack-Punkte werden in der Reihenfolge von der niedrigen zur hohen Version wie folgt markiert:

RPC_STATUS RPC_ENTRY RpcEpRegisterW (
    RPC_IF_HANDLE IfSpec,
    RPC_BINDING_VECTOR * BindingVector,
    UUID_VECTOR * UuidVector,
    RPC_WSTR Annotation
    )



RPC_STATUS RPC_ENTRY RpcServerRegisterIfEx (
    RPC_IF_HANDLE IfSpec,
    UUID * MgrTypeUuid,
    RPC_MGR_EPV * MgrEpv,
    vorzeichenlose int Flags,
    vorzeichenlose int MaxCalls,
    RPC_IF_CALLBACK_FN * IfCallback
    )



RPC_ENTRY RPC_STATUS RpcServerRegisterIf3 (
    RPC_IF_HANDLE IfSpec,
    UUID * MgrTypeUuid,
    RPC_MGR_EPV * MgrEpv,
    unsigned int Flags,
    unsigned int MaxCalls,
    unsigned int MaxRpcSize,
    RPC_IF_CALLBACK_FN * IfCallback,
    void * SecurityDescriptor
    )

Es gibt eine der offensichtlichsten Eigenschaften während des Registrierungsprozesses, die als der RPC-Komponentendienst identifiziert werden kann, den wir registrieren möchten: IfSpec-> InterfaceId.SyntaxGUID. Die in diesem Artikel beschriebene GUID war bisher dieselbe: 0x367ABB81, 0x9844, 0x35F1, 0xAD, 0x32, 0x98, 0xF0, 0x38, 0x00, 0x10, 0x03

Das ist {367ABB81-9844-35F1-AD32-98F038001003}.



Nach dem Hijacking einiger RPC-Funktionen, an denen wir interessiert sind, gibt es immer noch ein großes Problem, nämlich das Abrufen der Prozess- / Thread-Informationen des Clients

Glücklicherweise bietet RPCRT4.dll einige Funktionen, um leicht herauszufinden, wer der Client ist: I_RpcOpenClientThread, I_RpcOpenClientProcess.

Und die TOKEN-Gruppe von services.exe (sollte zu Admin gehören) gibt es kein Problem, TOKEN abzulehnen, aber es gibt keine Möglichkeit, zu einigen Gruppen mit geringen Berechtigungen zu wechseln, nur einige andere Methoden können verwendet werden ~



Beachten Sie bei der Verwendung von I_RpcOpenClientThread / I_RpcOpenClientProcess, dass DesiredAccess nicht zu hoch sein sollte. Welche Berechtigungen Sie schreiben müssen, im Allgemeinen QUERY_INFORMATION.



Der detaillierte Implementierungsteil wird nicht fortgesetzt, siehe Anhang.
