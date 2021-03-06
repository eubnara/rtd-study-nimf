======================================================
The Kerberos Protocol And Its Implementations
======================================================


- http://www.kerberos.org/software/tutorial.html
- https://zeroshell.org/kerberos/Kerberos-definitions/

위 문서를 번역하였습니다.

-----
목차
-----

1. 소개
2. 목표
3. 구성 요소와 용어의 정의

   1. Realm
   2. Principal
   3. Ticket
   4. Encryption

      1. Encryption type
      2. Encryption key
      3. Salt
      4. Key Version Number (kvno)

   5. Key Distribution Center (KDC)

      1. Database
      2. Authentication Server (AS)
      3. Ticket Granting Server (TGS)

   6. Session key
   7. Authenticator
   8. Replay Cache
   9. Credential Cache

4. Kerberos Operation

   1. Authentication Server Request (AS_REQ)
   2. Authentication Server Reply (AS_REP)
   3. Ticket Granting Server Request (TGS_REQ)
   4. Ticket Granting Server Replay (TGS_REP)
   5. Application Server Request (AP_REQ)
   6. Pre-Authentication [4.6 Application Server Replay (AP_REP) missing]

5. Ticket 자세히 살펴보기

   1. Initial tickets
   2. Renewable tickets
   3. Forwardable tickets

6. Cross Authentication

   1. Direct trust relationships
   2. Transitive trust relationships
   3. Hierarchical trust relationships


-------------
1. 소개
-------------


Kerberos 프로토콜은 개방되어 있고 안전하지 않은 네트워크 상에서도 신뢰할 수 있는 인증을 제공하기 위해 고안되었습니다. 이 네트워크에 속한 여러 호스트들 사이에서 오간 메시지들은 탈취될 수 있음을 가정하고 있습니다. 하지만, Kerberos 는 사용되는 컴퓨터들 자체가 취약한 상황에서는 어떠한 것도 보장하지 않습니다. 즉, 인증 서버, 애플리케이션 서버\(imap, pop, smtp, telnet, ftp, ssh, AFS, Ipr, ...\) 그리고 클라이언트들은 항상 최신성을 유지하여 요청하는 사용자와 서비스 제공자들의 진위성이 보장되어야 합니다.

위 내용은 한 문장으로 표현하면 다음과 같습니다.: **"Kerberos 는 신뢰할 수 없는 네트워크 상황에서 믿을 만한 호스트 간의 인증 프로토콜이다."** 예를 하나 들어 이 개념을 다시 설명해 보겠습니다. Kerberos 의 전략은 누군가가 서버에 접근 권한을 얻어서 비밀키가 포함된 파일을 복사할 수 있다면 아무짝에도 쓸모가 없습니다. 정말로, 침입자가 이 키를 다른 장비에 두고, 간단한 거짓 DNS 혹은 IP 를 얻기만 한다면 인증 서버에게 실제 사용자인 척 할 수 있습니다.


-------------
2. 목표
-------------

Kerberos 인증 시스템을 구성하고 있는 것들을 설명하고 동작과정을 살펴보기 전에 이 프로토콜이 얻고자 하는 목표들은 다음과 같습니다.

* 사용자의 비밀번호는 네트워크 상으로 오고 가면 안됩니다.
* 사용자의 비밀번호는 어떠한 형태로든 사용자 측 장비에 저장되어서는 안됩니다. 사용된 후에는 즉시 버려야 합니다.
* 사용자의 비밀번호는 인증 서버 데이터베이스라도 암호화되지 않은 형태로 저장되어서는 안됩니다.
* 사용자에게 작업 세션당 한번씩만 비밀번호를 입력하라고 해야 합니다. 따라서 같은 세션 동안 사용자들은 인증한 모든 서비스들에 접근할 때, 비밀번호를 다시 입력하지 않아도 됩니다. 이것은 **Single Sign-On** 이라는 것의 특징입니다.
* 인증 정보 관리는 인증 서버에서 중앙 집중화방식으로 이루어집니다. 애플리케이션 서버들은 사용자들의 인증 정보를 갖고 있어서는 안됩니다. 다음과 같은 효과를 얻기 위해 필수입니다.

  * 관리자는 한 곳에서 사용자 계정을 비활성화 할 수 있습니다. 여러 서비스를 제공하고 있는 여러 애플리케이션 서버들에 모두 작업을 하지 않아도 됩니다.
  * 한 사용자가 비밀번호를 바꾸면 모든 서비스들에 동시에 적용됩니다.
  * 불필요한 인증 정보의 중복 저장이 없습니다. 여러 곳에서 인증 정보를 저장하게 된다면 각각에서 안전장치를 두어야 합니다.

* 사용자들이 자신을 입증해야 하는 것 뿐만 아니라, 요청받은 애플리케이션 서버들도 사용자들에게 그들의 진위성을 입증해야 합니다. 이 특징은 **상호 인증\(Mutual authentication\)** 이라고 합니다.
* 인증과 권한 부여과정이 끝나면, 클라이언트와 서버는 필요하다면 암호화된 연결을 이루어야 합니다. 이를 위해서, Kerberos 는 데이터를 암호화하기 위한 암호화 키의 생성과 교환을 지원합니다.



---------------------------------
3. 구성 요소와 용어의 정의
---------------------------------


이 장은 대상과 용어들의 정의, 뒤에 Kerberos 프로토콜의 설명에 필수적인 배경지식을 설명합니다. 많은 정의들은 다른 것과 서로 얽혀있기 때문에, 가능하면 순서대로 설명해서 정의하기 전에 용어의 의미를 다루지 않겠습니다. 하지만, 모든 용어를 완전히 이해하기 위해서는 이 장을 두 번 읽어야 할 수도 있습니다.



3.1 Realm
===================

Realm 은 인증 관리 도메인을 지칭합니다. 이는 인증 서버가 사용자, 호스트 혹은 서비스를 인증할 수 있는 권위의 경계를 설정하기 위한 것입니다. 그렇다고 해서 사용자와 서비스 간의 인증이 반드시 같은 Realm 에 속해야만 하는 것은 아닙니다. 두 대상이 다른 Realm 에 속해 있지만 다른 Realm 간에 신뢰 관계가 있다면, 인증은 이루어질 수 있습니다. 이 특징은 **Cross-Authentication** 이라 불리고 아래에서 설명합니다.

기본적으로 사용자는(서비스는) 그 사람이(서비스가)  한 Realm 의 인증 서버와 비밀 (비밀번호 / 키) 를 공유하고 있다면, 그 Realm 에 속한 것입니다.

Realm 의 이름은 대소문자를 구분합니다. 대문자로 된 것과 소문자로 된 것에는 차이가 있습니다. 하지만 일반적으로 Realm 들은 대문자로 표현합니다. 한 조직에서 Realm 이름을 DNS 도메인과 같게 만드는 것 또한 좋은 관습입니다. (하지만 대문자로 말입니다.) Realm 명을 선정할 때, 방금 언급한 팁들을 따르는 것은 Kerberos 클라이언트 측 설정을 크게 단순화합니다. 특히 subdomain 들 사이에서 신뢰 관계를 형성해야 할 때 그렇습니다. 설정 예시를 들어보, 한 조직이 example.com 이라는 DNS 도메인에 속한다면, Kerberos realm 명은 EXAMPLE.COM 으로 하는 것이 적절합니다.



3.2 Principal
===================

Principal 은 인증 서버 데이터베이스에 존재하는 하나의 항목을 가리키는 이름입니다.
하나의 Principal 은 각 사용자, 호스트 혹은 해당 realm 의 서비스와 관련됩니다.
Kerberos 5 버전에서의 principal 은 다음과 같은 형식입니다.

::

    component1/component2/.../componentN@REALM

그러나 실제로는 최대 두 개의 component 이름만 사용됩니다.
한 사용자를 가리키는 항목은 다음 형식입니다.

::

    Name[/instance]@REALM

instance 는 선택사항이고 보통은 사용자가 어떤 타입인지를 잘 나타내기 위해 사용됩니다.
예를 들어 관리자는 보통 ``admin`` instance 를 갖게됩니다.
다음은 사용자들을 나타내는 principal 들의 예시입니다.

::

    pippo@EXAMPLE.COM  admin/admin@EXAMPLE.COM  pluto/admin@EXAMPLE.COM

서비스를 나타내는 항목들이라면, principal 은 다음 형태를 취할 겁니다.

::

    Service/Hostname@REALM

첫번째 component 는 서비스의 이름입니다. 예를 들면 ``imap``, ``AFS``, ``ftp`` 와 같은 이름입니다. 종종 ``host`` 라는 단어는 장비에 일반적인 접근을 나타낼 때 쓰입니다. (telnet, rsh, ssh) 두번째 component 는 요청된 서비스를 제공하는 장비의 완전한 hostname (FQDN) 을 나타냅니다. 이 component 는(FQDN) application server 의 ip 주소에 대한 DNS 역질의와 정확히 일치해야 합니다. 다음은 서비스를 가리키는 principal 들의 유효한 예입니다.

::

    imap/mbox.example.com@EXAMPLE.COM
    host/server.example.com@EXAMPLE.COM
    afs/example.com@EXAMPLE.COM

위에서 언급한 예시 중, 마지막 경우는 예외입니다. 두번째 component 가 hostname 이 아니라 principal 이 가리키는 AFS cell 의 이름이기 때문입니다.
마지막으로, 사용자나 서비스를 가리키지는 않지만 인증 시스템의 동작과정에서 필요한 principal 들이 있습니다. 전체 예제는 ``krbtgt/REALM@REALM``, 그리고 이것과 연관된 키가 Ticket Granting Ticket 을 암호화하는데 사용된다는 것입니다. (나중에 살펴보겠습니다.)

Kerberos 4 버전에서는 2개보다 많은 component 를 가질 수 없고, ``/`` 대신에 ``.`` 으로 component 가 구분됩니다. 또한, 서비스를 가리키는 principal 에서 hostname 은 FQDN 이 아니라 짧은 형태입니다. (PQDN) 다음은 Kerberos 4 에서 유효한 예입니다.

::

    pippo@EXAMPLE.COM  pluto.admin@EXAMPLE.COM  imap.mbox@EXAMPLE.COM


3.3 Ticket
===================

티켓은 애플리케이션 서버에 자신의 신원의 진위성을 입증하기 위해 클라이언트가 주는 것입니다. 티켓은 인증 서버에 의해 발행되며 티켓이 의도한 서비스의 비밀 키를 이용해 암호화 됩니다. 그 비밀키는 인증 서버와 서비스를 제공하는 서버만이 공유하는 비밀이기 때문에 티켓을 요구한 클라이언트 조차도 알 수 없으며 암호화된 내용을 바꿀 수도 없습니다. 티켓에 들어간 주요한 내용은 다음을 포함합니다.

- 요구한 사용자의 principal (일반적으로 사용자계정)
- 요구된 서비스의 principal
- 티켓이 사용될 client 장비의 ip 주소. Kerberos 5 버전에서 이 항목은 선택적이고 NAT 나 multihomed 환경에서의 클라이언트를 위해 여러 개의 값이 될 수도 있습니다.
- 티켓 유효성이 시작된 시간과 날짜 (timestamp 형식으로)
- 티켓의 최대 수명
- 세션 키 (이건 아래 설명할텐데, 근본적인 역할을 합니다.)

각 티켓은 만료시기가 있습니다. (일반적으로 10시간 입니다.) 이 점은 중요한데, 인증서버는 이미 발행된 티켓에 대해서 더 이상 아무런 제어를 할 수 없기 때문입니다.
realm 관리자가 특정 사용자에 대해서 아무때나 더 이상 티켓을 발행하지 못하도록 막을 수 있더라도, 이미 소유한 티켓을 사용하는 걸 막을 순 없습니다. 이것이 제한시간을 초과하여 남용하는 것을 막기 위해 티켓의 수명을 제한하는 이유입니다.

티켓은 많은 정보와 행위를 특징짓는 flag 들을 갖고 있습니다. 하지만 그 내용을 지금 살펴보진 않을 것입니다. 인증 시스템이 어떻게 동작하는지 보고나서 티켓과 flag 들을 다시 살펴보겠습니다.


3.4 암호화
===================

알다시피, Kerberos 는 인증과정에서 여러 참여자들 간에 오고가는 메시지들을 암호화하고 복화하하는 것이 종종 필요합니다. Kerberos 는 대칭키 암호화 방법만 사용한다는 것을 주목해야 합니다. (즉, 같은 키로 암호화와 복호화를 한다는 것입니다.) 어떤 프로젝트는 (pkinit 과 같은) 확인된 공개 키에 대응하는 비밀 키의 제출을 통해 초기 사용자 인증 과정을 할 수 있는 공개 키 시스템 도입을 진행 중입니다. 하지만 표준화가 되지 않았으며 그에 대한 설명은 현재 생략하겠습니다.

3.4.1 암호화 종류
---------------------------

Kerberos 4 는 56비트의 DES 암호화 한 종류만 구현했습니다. 이 암호화의 취약함과 더불어 다른 프로토콜의 취약함으로 인해 Kerberos 4 는 더 이상 사용되지 않습니다.
Kerberos 5 에서는 지원되는 암호화 방법들의 종류나 수를 미리 정하지 않습니다. 다양한 암호화를 지원하고 최선을 선택하는 것은 각 구현의 역할입니다.
그러나, 이 프로토콜의 유연성과 확장성은 Kerberos 5 의 다양한 구현 사이에서 상호운용성 문제들을 심화시킵니다. 다른 구현을 사용하는 클라이언트들, 애플리케이션 서버들 그리고 인증 서버들이 상호운용되려면 적어도 하나의 암호화 타입을 공통으로 갖고 있어야 합니다. 이와 관련한 한 고전 예제는 Kerberos 5 의 유닉스 구현과 윈도우의 Active Directory 에서 쓰이는 것과의 상호운용의 어려움입니다.
정말로, Windows Active Directory 는 제한된 숫자의 암호화 방식만 지원하며 유닉스와 56 비트의 DES 암호화 방식만 공통됩니다. 상호운용성이 보장되어야 한다면 위험성을 알고 있음에도 후자가 유지되어야 했습니다. 이 문제는 그후 MIT Kerberos 5 의 1.3 버전에서 해결되었습니다. 이 버전에서는 RC4-HMAC 암호화 기법을 지원하며 Windows 에도 존재하고 DES 보다 안전합니다. 지원되는 암호화 기법 중에서 (윈도우에서는 아닙니다.) triple DES (3DES), 더 새로운 AES128, AES256 은 언급될 만한 가치가 있습니다.



3.4.2 암호화 키
---------------------------

위에서 언급한 것처럼, 인증 서버의 데이터베이스에서라도 사용자의 비밀번호를 암호화되지 않은 형태로 저장하는 것을 막는 것이 Kerberos 프로토콜의 목표 중 하나입니다. 각 암호화 알고리즘들은 그것만의 고유한 키 길이를 사용한다는 점을 감안하면, 사용자들이 지원되는 각 암호화 방법의 고정된 길이로 다른 비밀번호를 사용하라고 강요하지 않는다면, 암호화 키는 비밀번호로 할 수 없다는 것이 명백합니다.
이러한 이유로 ``string2key`` 함수가 나왔습니다. 이것은 암호화되지 않은 비밀번호를 사용될 암호화 형식에 맞는 암호화 키로 바꿔줍니다. 이 함수는 사용자가 비밀번호를 변경하거나 입력할 때마다 호출됩니다. ``string2key`` 는 해시 함수입니다. 역으로 할 수 없다는 뜻입니다. 암호화 키로는 이를 생성한 비밀번호가 무엇인지 알 수 없습니다. (brute force 방법이 아니라면) 유명한 해시 알고리즘은 MD5, CRC32 가 있습니다.



3.4.3 Salt
---------------------------

Kerberos 4 와 달리 Kerberos 5 에서는 비밀번호 ``salt`` 개념이 도입되었습니다. 이것은 키를 얻기 위해 ``string2key`` 함수가 적용되기 전 암호화되지 않은 비밀번호에 붙는 문자열입니다. Kerberos 5 에서는 사용자의 principal 과 같은 문자열을 ``salt`` 로 사용합니다.

.. math::

    K_{pippo} = string2key ( P_{pippo} + "pippo@EXAMPLE.COM" )

:math:`K_{pippo}` 는 ``pippo`` 사용자의 암호화 키이고 :math:`P_{pippo}` 는 사용자의 암호화되지 않은 비밀번호입니다.
이러한 형식의 salt 는 다음과 같은 이점을 갖습니다.

- 같은 Realm 에 속한 두 개의 principal 이 있고, 암호화되지 않은 비밀번호가 같더라도 암호화 키는 달라집니다. 예를 들어, 관리자가 일상적인 일을 위한 principal (pippo@EXAMPLE.COM) 을 하나 갖고 있고, 관리자의 업무를 위한 principal (pippo/admin@EXAMPLE.COM) 을 하나 갖고 있다고 상상해 봅시다. 이 사용자는 편의성을 위해서 두 개의 principal 에 대해서 같은 비밀번호를 설정할 가능성이 높습니다. salt 로 인하여 관련된 키들이 다를 수 있도록 보장할 수 있습니다.
- 한 사용자가 다른 realm 에 두 개의 principal 이 있다면, 두 realm 모두에 대해서 암호화되지 않은 비밀번호가 같을 확률이 높습니다. salt 의 존재 덕분에 한 realm 에 계정이 손상되더라도 이게 곧바로 다른 쪽의 계정을 손상시키진 않습니다.

Kerberos 4 와의 호환성을 위해서 null salt 을 설정할 수도 있습니다. 반대로 AFS 와의 호환성을 위해서 principal 의 완전한 이름이 아니라 단순히 cell 의 이름으로 salt 를 설정할 수도 있습니다.

암호화 종류, ``string2key``, ``salt`` 의 개념을 살펴보았으므로 다음 생각이 맞다는 걸 알 수 있습니다. '다양한 Kerberos 구현 간에 상호운용성이 있기 위해서는 공통된 암호화 종류를 중재하는 것만으로는 충분하지 않습니다. 같은 종류의 ``string2key``, ``salt`` 또한 필요합니다.'

``string2key``, ``salt`` 의 개념을 설명하는데 있어서 서버들의 principal 에 대해서는 언급이 없고 사용자 principal 에 대해서만 말했다는 걸 주목합니다. 이유는 명백합니다. 서비스는 인증서버와 비밀을 공유하더라도 암호화되지 않은 비밀번호가 아니고, (누가 그걸 입력하겠습니까?) Kerberos 서버에서 관리자가 한번 생성한 키는 있는 그대로 서비스를 제공하는 서버에 저장됩니다.


3.4.4 Key Version Number (kvno)
------------------------------------------------------

사용자가 비밀번호를 바꾸거나 관리자가 애플리케이션 서버의 비밀 키를 변경하였을 때, 카운터를 하나 올려서 이 변경을 기록합니다. 키 버전을 식별하기 위한 카운터의 현재 값은 Key Version Number 이고, 간략하게는 ``kvno`` 라고 불립니다.


3.5 Key Distribution Center (KDC)
===============================================

인증서버에 대해 일반화하여 언급하였습니다. 사용자와 서비스의 인증에 참여한 중요한 대상이기 때문에 한층 더 깊게 살펴보겠습니다. 인증 과정의 모든 세부 사항을 살펴보진 않습니다. 이에 대한 내용은 프로토콜 동작과정을 다룬 부분에서 살펴봅니다.

서비스에 접근을 위한 티켓 분배 기능에 기반한 Kerberos 환경의 인증서버는 Key Distribution Center 로 불리며 더 간략하게 KDC 라고 불립니다.
KDC 는 모두 하나의 물리 장비에 있기 때문에, (종종 하나의 프로세스와 일치합니다.) 논리적으로 세 개의 부분으로 나뉜 것으로 고려됩니다.: 데이터베이스, Authentication Server (AS), Ticket Granting Server(TGS). 이것들을 한번 살펴봅시다.


.. Note:: 한 realm 에서 Master/Slave 방식 (MIT and Heimdal) 혹은 Multimaster 방식 (Windows Active Directory) 으로 여분의 서버를 두는 것이 가능합니다. 어떻게 중복성을 얻는지에 대해서는 프로토콜에 명시되어 있지 않고 사용되는 구현에 따라 다릅니다. 여기서 이에 대한 내용을 언급하진 않습니다.


3.5.1 데이터베이스
-------------------------

데이터베이스는 사용자, 서비스들과 연관된 항목들을 위한 컨테이너입니다. ``principal`` 이라는 용어는 종종 ``entry`` 의 동의어로 사용될지라도, 우리는 principal 을 사용해서 entry 를 가리키겠습니다. (i.e. entry 의 이름) 각각의 entry 는 다음 정보를 갖고 있습니다.

- entry 와 연관된 principal
- 암호화 키, 관련 kvno
- 해당 principal 과 관련된 티켓의 최대 유효 기간
- 해당 principal 과 관련된 티켓이 갱신될 수 있는 최대 시간 (Kerberos 5 에서만)
- 티켓의 행위를 특징지을 수 있는 특성과 플래그
- 비밀번호 만료 날짜
- principal 만료 날짜, 이 이후엔 티켓이 만들어지지 않습니다.

| 데이터베이스에 있는 키를 훔치기 어렵게 하기 위해서, 구현된 방법들은 ``마스터 키`` 를 사용해서 데이터베이스를 암호화합니다. ``마스터 키`` 는 ``K/M@REALM`` 과 연관됩니다. 백업이나 KDC master 에서 slave 로 내용 전파할 때 사용되는 데이터베이스 덤프에서 조차도 이 키를 이용해서 암호화됩니다. 데이터베이스 덤프 내용을 다시 넣기 위해서 마스터 키를 알아야 합니다. ``마스터 키`` 는 ``K/M@REALM`` principal 과 연관됩니다.
| 백업으로 혹은 KDC 마스터에서 슬레이브로 전파를 위한 어떠한 데이터베이스 덤프라도 이 키를 이용하여 암호화합니다.
| 덤프 내용을 리로딩하기 위해서는 ``마스터 키`` 가 필요합니다.


3.5.2 인증 서버 (Authentication Server, AS)
--------------------------------------------------

인증 서버는 KDC 의 한 부분이고, 사용자로부터의 첫 요청에 응답합니다. 사용자는 아직 인증되지 않았다면 비밀번호를 입력해야 합니다. 인증 요청의 응답으로 인증 서버는 ``Ticket Grating Ticket`` 이라는 특별한 티켓을 발급합니다. 줄여서 ``TGT`` 라고 부릅니다. 여기에 연관된 principal 은 ``krbtgt/REALM@REALM`` 입니다. 만일 요청한 사용자들이 실제로 그들이 말한 대로 그들이 맞다면 (그들이 어떻게 이를 보여줄 수 있는지는 나중에 살펴보겠습니다.) 그들은 다시 비밀번호를 입력할 필요 없이 ``TGT`` 를 이용해서 다른 서비스 티켓을 얻을 수 있습니다.

3.5.3 티켓 배포 서버 (Ticket Granting Server, TGS)
--------------------------------------------------

티켓 배포 서버(TGS)는 유효한 ``TGT`` 를 갖고 있는 사용자들에게 서비스 티켓을 배포하는 KDC 컴포넌트입니다. 애플리케이션 서버들에 요청한 자원들을 얻으려는 정체의 진위 여부를 보장합니다. TGS 는 하나의 애플리케이션 서버라고 볼 수 있습니다. (여기에 접근하기 위해서는 TGT 를 제출해야 한다는 점에서 그렇습니다.) 서비스 티켓을 발급하는 서비스를 제공합니다. ``TGT`` 와 ``TGS`` 라는 축약어를 헷갈리면 안됩니다. ``TGT`` 는 티켓을 가리키고 ``TGS`` 는 서비스를 가리킵니다.



3.6 세션 키
====================

앞에서 본 바와 같이 사용자들과 서비스들은 KDC 와 비밀을 공유합니다. 사용자 측면에서 그 비밀은 비밀번호로부터 유래한 키입니다. 한편 서비스 측면에서는 그것들의 비밀 키입니다. (운영자가 세팅합니다.) 이러한 키들은 작업 세션이 변할 때 변하지 않기 때문에 장기간(long term) 입니다.

하지만 사용자는 적어도 서버와 작업 세션을 갖는 동안은 서비스와 비밀을 공유하는 것도 필요합니다: 티켓이 발급될 때 KDC 가 만들어주는 이 키는 ``세션 키(Session Key)`` 라고 부릅니다. 서비스를 위한 세션키 한 부는 KDC 가 티켓에 동봉합니다. (어떠한 경우라도 애플리케이션 서버는 장기간(long term) 키를 알고 복호화하여 세션키를 추출할 수 있습니다.) 사용자를 위한 다른 세션키 한 부는 사용자의 장기간 키로 암호화된 패킷에 캡슐화됩니다. 세션 키는 사용자의 진위 여부를 가릴 때 중요한 역할을 합니다. 이는 다음 단락에서 살펴보겠습니다.


3.7 인증자 (Authenticator)
========================================

사용자 principal 이 티켓에 있고 애플리케이션 서버만이 그러한 정보를 추출하고 관리까지 할 수 있다 하더라도 (이 티켓은 서비스의 비밀 키로 암호화되기 때문입니다.) 사용자의 진위를 보장하기에는 충분하지 않습니다. 한 사기꾼이 합법적인 사용자가 애플리케이션 서버로 보낸 티켓을 탈취할 수 있습니다. (개방되고 안전하지 않은 네트워크 상이라는 가정을 기억하십시오.) 그리고 적당한 시간에 이를 보내 서비스를 불법적으로 얻을 수 있습니다. 한편 장비의 IP 주소들을 넣는 것은 유용하지 않습니다: 개방되고 안전하지 않은 네트워크에서는 주소들은 쉽게 위조될 수 있습니다. 이 문제를 해결하기 위해 다음 사실을 이용해야 합니다. 사용자와 서버는 적어도 세션동안은 세션 키를 그들만 알고 있습니다. (KDC 도 또한 세션 키를 만들었기 때문에 알고 있습니다. 하지만, KDC 는 정의대로 신뢰할 수 있습니다.) 따라서, 다음 전략이 적용됩니다: 티켓을 포함하는 요청에서 사용자는 다른 패킷(인증자, Authenticator)을 추가합니다. 여기에는 사용자 principal 과 타임스탬프 (그 당시) 가 포함되어 있고 세션 키로 암호화됩니다: 서비스를 제공해야 하는 서버는 이 요청을 받자마자 첫번째 티켓을 열어 세션 키를 추출합니다. 만일 그 사용자가 정말 맞다면, 서버는 인증자를 복호화하고 타임스탬프를 추출할 수 있습니다. 만일 추출한 타임스탬프가 2분 이하로 (설정으로 더 여유를 줄 수 있습니다.) 서버시간과 다르다면, 인증은 성공합니다. 이는 같은 Realm 에 속해있는 장비들 간 동기화가 중요하다는 걸 말해줍니다.


3.8 Replay cache
======================


사기꾼이 티켓과 인증자를 동시에 훔치고 인증자가 유효한 2분 동안 사용할 가능성은 존재합니다. 이것은 많이 어렵지만 불가능하진 않습니다. Kerberos 5 에서 이 문제를 해결하기 위해서 ``Replay cache`` 가 등장했습니다. 애플리케이션 서버들에서 (TGS 에서도 또한) 2분 내에 도착한 인증자를 기억하고 만일 복제품이라면 인증자를 거부할 수 있는 공간이 존재합니다. 사기꾼이 티켓과 인증자를 복사하고 이것들을 애플리케이션 서버에 합법적인 요청이 도착하기 전에 도달하게 할 정도로 똑똑하지 않다면, 이 공간으로 문제는 해결됩니다. 이 상황은 정말 날조입니다. 진짜 사용자는 거부되고 사기꾼이 서비스에 접근할 수 있기 때문입니다.


3.9 Credential Cache
========================

클라이언트 측은 절대로 사용자의 비밀번호를 보관하지 않고 ``string2key`` 를 적용하여 얻어진 비밀 키를 기억하지도 않습니다: 이것들은 KDC 로부터의 응답을 복호화하고 곧바로 버려집니다. 그러나 한편 작업 세션당 사용자가 한번만 비밀번호를 입력하면 되는 싱글사인온(SSO) 기능을 구현하기 위해서 티켓 그리고 관련된 세션 키를 기억해야 합니다. 이 데이터가 저장되는 장소를 ``Credential Cache`` 라고 부릅니다. 이 캐시가 저장되는 곳은 프로토콜에 의존해서는 안되고 구현 방식에 따라 달라야 합니다. 종종 호환성 목적으로 파일시스템에 위치합니다.(``MIT`` 와 ``Heimdal``) 또 다른 구현체에서는 (AFS 와 Active Directory) 취약한 클라이언트의 이벤트들에 보안성을 높이기 위해서 ``Credential cache`` 가 커널만 접근 가능한 메모리 영역에 위치하고 디스크로 스왑되지 않습니다.


----------------------------
4. Kerberos Operation
----------------------------

마침내 앞선 장들에서 설명한 개념들을 습득했다면, 커버러스가 어떻게 동작하는지 논의할 수 있습니다. 인증과정 중 클라이언트와 KDC 사이, 클라이언트와 애플리케이션 서버 사이에서 오고 가는 각 패킷들을 나열하고 설명해 보면서 진행하겠습니다. 여기서 기억해야할 중요한 점은 애플리케이션 서버는 절대로 KDC 와 직접적으로 통신하지 않는다는 점 입니다: 서비스 티켓은 TGS 에 의해 패킷화되었더라도 서비스에 이를 사용하고 싶어하는 사용자를 통해 도달합니다. 우리가 논의할 메시지들은 다음과 같습니다. (그 아래 그림도 함께 보십시오.)

- ``AS_REQ`` 는 초기 사용자 인증 요청입니다. (i.e. ``kinit`` 으로 생성됩니다.) 이 메시지는 인증서버(AS, Authentication Server) 로 알려진 KDC 컴포넌트로 갑니다.
- ``AS_REP`` 는 이전 요청에 대한 인증서버의 응답입니다. 기본적으로 이것은 ``TGT`` (``TGS`` 비밀 키로 암호화되어 있습니다.) 와 세션 키(요청한 사용자의 비밀 키로 암호화되어 있습니다.) 를 포함합니다.
- ``TGS_REQ`` 는 클라이언트에서 티켓 발급 서버(TGS) 로 서비스 티켓을 위한 요청입니다. 이 패킷은 이전 메시지에서 얻은 ``TGT`` 와 클라이언트에 의해 만들어진 인증자를 포함하고 세션 키로 암호화됩니다.
- ``TGS_REP`` 는 티켓 발급 서버(TGS) 의 이전 요청에 대한 응답입니다. 이 응답 안에는 요청한 서비스 티켓 (해당 서비스의 비밀 키로 암호화되어 있습니다.) 과 티켓 발급 서버(TGS) 에 의해 발급된 서비스 세션 키가 있고 인증서버로부터 발급받은 이전 세션키를 사용하여 암호화됩니다.
- ``AP_REQ`` 는 서비스에 접근하기 위해서 클라이언트가 애플리케이션 서버로 보내는 요청입니다. 구성요소들은 티켓 발급 서버(TGS) 로부터 얻은 서비스 티켓과 이전 응답 그리고 클라이언트가 생성한 인증자 입니다. 그러나 이번엔 서비스 세션 키를 이용하여 암호화합니다. (서비스 세션 키는 티켓 발급 서버(TGS) 에 의해 생성되었습니다.)
- ``AP_REP`` 는 애플리케이션 서버가 클라이언트에게 자신이 클라이언트가 기대한 그 서버라고 증명하기 위해 보내는 응답입니다. 이 패킷은 항상 요청되진 않습니다. 클라이언트는 상호인증이 필요할 때에만 서버에 이 요청을 합니다.

.. image:: images/krbmsg.gif

이제 각각의 이전 단계들을 Kerberos 5 를 참고하여 그러나 버전 4 와는 차이를 보며 더욱 자세히 살펴보겠습니다. 하지만 Kerberos 프로토콜은 꽤 복잡하고 이 문서는 정확한 동작 세부사항을 알고 싶어하는 사람들을 위한 가이드로 작성된 것은 아니라는 점을 명심해야 합니다. (어쨌든 이것들은 RFC1510 에 이미 써있습니다.) 아래 논의들은 의도적으로 추상적으로 되어있습니다. 하지만 KDC 로그들을 조사하여 다양한 인증 전환들과 발생하는 문제들을 이해하려는 사람들에게는 충분합니다.

.. Note::

    다음 장들에서 암호화되지 않은 데이터는 소괄호 ( ``(``, ``)`` ) 로 감쌉니다. 그리고 암호화된 데이터는 중괄호 ( ``{``, ``}`` ) 로 감쌉니다: ``( x, y, z )`` 는 x, y, z 가 암호화되지 않았다는 걸 의미합니다; ``{ x, y, z}K`` 는 x, y, z 가 모두 대칭키 K 로 암호화되었다는 걸 나타냅니다. 패킷 안에 나열된 컴포넌트들이 실제 메시지 (UDP 나 TCP) 에서 발견되는 순서와는 관련이 없다는 점도 주목해야 합니다. 이 논의는 매우 추상적입니다. 더 자세한 세부사항을 원한다면 RFC1510 을 참고하세요. 이는 묘사적인 프로토콜 ASN.1 에 대한 좋은 배경을 갖고 있습니다.


4.1 Authentication Server Reqeust (AS_REQ)
===================================================

초기 인증 요청으로 알려진 단계에서 클라이언트는 KDC 에게 (더 구체적으로는 AS) Ticket Granting Ticket 을 요청(kinit) 합니다. 이 요청은 완전히 복호화되고 다음과 같이 생겼습니다.

.. math::

    AS\_REQ = ( Principal_{client}, Principal_{service}, IP\_list, Lifetime )

- :math:`Principal_{client}`: 인증을 원하는 사용자와 연관된 principal (e.g. pippo@EXAMPLE.COM)
- :math:`Principal_{service}`: 이 티켓이 목표로 하는 서비스와 연관된 principal 입니다. 따라서, "krbtgt/REALM@REALM" 이라는 문자열입니다. (**주석** * 참고)
- :math:`IP\_list`: 발행될 티켓이 사용가능한 호스트를 나타내는 IP 주소 목록입니다. (**주석** ** 참고 )
- :math:`Lifetime`: 발행된 티켓이 유효한 최대 시간을 나타냅니다.

**주석** * : 초기 인증 요청에 :math:`Principal_{service}` 를 넣는 것은 불필요해 보일지도 모릅니다. 일관적으로 TGS principal 을 넣기 때문입니다. ( 예) *krbtgt/REALM@REALM* ) 그러나 아닌 경우도 있습니다. 어떤 사용자가 작업 세션동안 단 하나의 서비스만 사용하려고 할 때는 싱글사인온(SSO) 를 사용하지 않게 되고 AS 에게 직접 그 서비스를 위한 티켓을 요청하게 됩니다. 따라서, TGS 로의 요청은 생략하게 됩니다. 운영관점에서 (MIT 1.3.6), 다음 명령으로 충분합니다. ``kinit -S imap/mbox.example.com@EXAMPLE.COM pippo@EXAMPLE.COM``

**주석** ** : :math:`IP\_list` 값이 null 일 수도 있습니다. 이런 경우 상응하는 티켓은 어느 장비에서나 사용될 수 있습니다. 이는 NAT 하에 있는 사용자들의 문제를 해결합니다. 서비스에 도달하는 요청에 있는 원 주소는 요청한 사용자의 주소와는 다르고 NAT 를 만든 라우터 주소와 같기 때문입니다. 대신에 하나 이상의 네트워크 카드를 갖고 있는 장비들에 대해서 :math:`IP\_list` 에는 모든 카드들에 대한 ip 주소를 포함해야 합니다. 서비스를 제공하는 서버와 어떤 것이 연결될 것인지 사전에 예측하기 어렵기 때문입니다.


4.2 Authentication Server Reply (AS_REP)
=============================================

이전 요청이 도달하였을 때, AS (인증 서버) 는 KDC 데이터베이스에 :math:`Principal_{Client}` 와 :math:`Principal_{Service}` 가 존재하는지를 확인합니다.: 둘 중에 하나라도 존재하지 않는다면 에러메시지가 클라이언트에 전달되고 그렇지 않으면 다음과 같이 AS 가 응답을 처리합니다.:

- 세션 키를 무작위로 만듭니다. 이를 클라이언트와 TGS 가 공유합니다. :math:`SK_{TGS}` 라 부르겠습니다.
- 요청한 사용자의 principal, 서비스 principal (보통은 krbtgt/REALM@REALM 입니다만, 이전 절에 **주석** * 을 참고하세요.), IP 주소 목록 (이 세 개의 정보들은 AS_REQ 패킷으로 전달된 그대로 복사됩니다.), timestamp 형식의 날짜와 시간 (KDC 의), lifetime (아래 **주석** * 참고) 그리고 마지막으로 세션키 (:math:`SK_{TGS}`)를 안에 넣어 Ticket Granting Ticket 을 만듭니다.: Ticket Granting Ticket 은 다음과 같습니다.

.. math::

    TGT = ( Principal_{Client} , krbtgt/REALM@REALM , IP\_list , Timestamp , Lifetime , SK_{TGS} )

- 다음을 포함하는 응답을 생성하여 전송합니다.: 서비스에 대한 암호 키로 암호화한 이전에 만들어진 티켓(:math:`K_{TGS}` 라 부르겠습니다.); 서비스를 요청한 사용자의 암호 키로 암호화한 서비스 principal, timestamp, lifetime 그리고 세션 키. 요약하면:

.. math::

    AS\_REP = \{ Principal_{Service} , Timestamp , Lifetime , SK_{TGS} \}K_{User}\ \ \{ TGT \}K_{TGS}

이 메시지에 불필요한 정보가 포함된 것처럼 보일 수 있습니다. (:math:`Principal_{Service}`, timestamp, lifetime 그리고 세션 키) 그러나 아닙니다.: TGT 안에 포함된 정보들은 서버의 암호 키로 암호화되었기 떄문에 클라이언트가 읽을 수 없고 반복되어야 합니다. 여기서 클라이언트가 응답 메시지를 받았을 때 사용자에게 암호를 입력하라고 요청합니다. salt 가 비밀번호에 붙여지고 ``string2key`` 함수가 적용됩니다.: 이 키로 KDC 에 의해 데이터베이스에 저장된 사용자의 암호 키로 암호화된 메시지의 일부분을 복호화하는 시도를 합니다. 사용자가 정말 맞다면, 즉 정확한 암호를 입력했다면, 복호화 과정은 성공하고 세션 키를 추출하고 TGT (암호화된 상태 그대로) 와 함께 사용자의 credential cache 에 저장됩니다.

**주석** * : 실제 lifetime, 예) 티켓에 들어가는 값은 다음 값들 중 가장 낮은 값입니다.: 사용자에 의해 요청된 lifetime, 사용자 principal 에 포함된 것 혹은 서비스 principal 에 있는 것. 실제로 구현의 관점에서 KDC 의 설정으로부터 다른 제한이 설정될 수 있고 아무 티켓에 적용될 수 있습니다.


4.3 Ticket Granting Server Request (TGS_REQ)
===================================================

이 시점에서 자신을 증명한 사용자는 (따라서 사용자의 credential cache 에는 TGT 와 세션 키 :math:`SK_{TGS}` 가 있고 서비스에 접근을 하고 싶으나 아직 알맞는 티켓을 갖고있지 않을 때) Ticket Granting Service 에 요청(TGS_REQ) 을 다음과 같이 구성하여 보냅니다.:

- 사용자 principal, 클라이언트 장비 timestamp 로 인증자를 만들고 TGS 와 공유하고 있는 세션 키로 모두 암호화합니다. 예:

.. math::

    Authenticator = \{\ Principal_{Client}\ ,\ Timestamp\ \}SK_{TGS}

- 다음을 포함하는 요청 패킷을 만듭니다.: 필요로 하는 티켓에 대한 서비스 principal 과 복호화된 lifetime; TGS 의 키로 이미 암호화된 Ticket Grating Ticket; 그리고 방금 만든 인증자.
요약하면:

.. math::

    TGS\_REQ = ( Principal_{Service} , Lifetime,  Authenticator )\ \{\ TGT\ \}K_{TGS}


4.4 Ticket Granting Server Replay (TGS_REP)
==================================================

이전 요청이 도착하면 TGS 는 먼저 요청한 서비스의 principal(:math:`Principal_{Service}`) 가 KDC 데이터베이스에 존재하는지를 확인합니다.: 존재한다면, krbtgt/REALM@REALM 의 키로 TGT 을 열고 세션 키를(:math:`SK_{TGS}`) 추출한 다음 인증자를 복호화하는데 씁니다. 발행될 서비스 티켓에 대해서 다음 조건들을 충족하는지 확인합니다.

- TGT 가 만료되지 않았다.
- :math:`Principal_{Client}` 가 인증자에 존재하고 TGT 안에 있는 것과 일치한다.
- 인증자가 replay cache 에 존재하지 않고 만료되지 않았다.
- :math:`IP\_list` 가 null 이 아니라면, 요청 패킷의(TGS_REQ) 원 주소가 이 목록에 포함되어 있는 것들 중 하나인지 확인한다.

위와 같이 확인된 조건들은 TGT 가 정말 요청한 사용자의 것이라는 걸 증명합니다. 이에 TGS 는 다음과 같은 응답을 처리합니다.:

- 클라이언트와 서비스 간에 공유하는 비밀이 될 세션 키를 무작위로 생성한다. 이를 :math:`SK_{Service}` 라 하자.
- 요청한 사용자의 principal, 서비스 principal, IP 주소들, timestamp 형식의 날짜와 시간, lifetime (TGT 의 lifetime 과 연관된 서비스 principal 의 lifetime 중 최소값) 그리고 마지막으로 세션 키 :math:`SK_{Service}` 를 안에 넣어 서비스 티켓을 만든다. :math:`T_{Service}` 로 알려진 새로운 티켓은 다음과 같다.

.. math::

    T_{Service} = ( Principal_{Client} , Principal_{Service} , IP\_list , Timestamp , Lifetime , SK_{Service} )

- 다음을 포함하는 응답을 보낸다.: 서비스 비밀 키(:math:`K_{Service}` 라 부르자.) 로 암호화한 방금 생성한 티켓; TGT 에서 추출한 세션 키를 사용하여 모두 암호화된 서비스 principal, timestamp, lifetime 그리고 새로운 세션 키. 요약하면 다음과 같다.


.. math::

    TGS\_REP = \{ Principal_{Service} , Timestamp , Lifetime , SK_{Service} \} SK_{TGS}\ \ \{ T_{Service} \} K_{Service}



..

    클라이언트 캐시에 세션 키(:math:`SK_{TGS}`) 를 가진 클라이언트가 응답을 수신하였을 때, 클라이언트는 다른 세션 키를 포함하는 메시지를 복호화하고 이를 서비스 티켓(:math:`T_{Service}`, 암호화된 채로 있음) 과 함께 저장한다.


4.5. Application Request (AP_REQ)
=======================================

서비스에 접근하기 위한 자격 증명을 (티켓과 세션 키) 갖고 있는 클라이언트는 애플리케이션 서버에 AP_REQ 메시지를 통해 자원 접근을 요청합니다. 다음을 염두에 두어야 합니다. KDC 가 참여한 이전의 메시지들과 달리 AP_REQ 는 표준이 아니고 애플리케이션에 따라 다릅니다. 따라서, 클라이언트가 자신의 정체를 서버에 증명하기 위한 자격 증명을 사용하는데, 애플리케이션 프로그래머는 어떻게 자격 증명을 할지에 대한 전략을 세우는 역할을 합니다. 그러나, 우리는 다음 예와 같은 전략을 생각해 볼 수 있습니다.

- 클라이언트는 사용자 principal 과 timestamp 를 포함하는 인증자를 만들고 모두 애플리케이션 서버와 공유하는 세션 키 :math:`SK_{Service}` 로 암호화합니다. 예:

.. math::

    Authenticator = \{\ Principal_{Client}\ ,\ Timestamp\ \}SK_{Service}

- 클라이언트는 서비스의 키로 암호화된 서비스 티켓 :math:`T_{Service}` 과 방금 만든 인증자를 포함하는 요청 패킷 만듭니다. 요약하면:

.. math::

    AP_{REQ} = Authenticator\ \{\ T_{Service}\ \}K_{Service}

이전 요청이 도달하면, 애플리케이션 서버는 요청된 서비스의 암호 키를 사용하여 열고, 인증자를 복호화할 때 쓰이는 세션 키 :math:`SK_{Service}` 를 추출합니다. 요청한 사용자의 진위를 파악하고 서비스에 접근할 수 있도록 서버는 다음 조건들을 확인합니다.:

- 티켓이 만료되지 않았다.
- 인증자에 포함된 :math:`Principal_{Client}` 가 티켓에 있는 것과 일치한다.
- 인증자가 replay cache 에 존재하지 않고 만료되지 않았다.
- :math:`IP\_list` (티켓에서 추출된) 가 null 이 아닐 때, 요청 패킷(AP_REQ)의 원 주소가 이 목록 중에 있다.

**주석**: 방금 전략은 Ticket Granting Server 가 서비스 티켓을 요청하는 사용자의 진위를 파악하는데 쓰는 전략과 유사합니다. 놀랄일은 아닙니다. 이미 설명하기로 TGS 는 하나의 애플리케이션 서버로 생각할 수 있고, TGT 로 그들의 정체를 증명하는 이들에게 티켓을 발급하는 서비스를 제공합니다.

4.6 Pre-Authentication
===============================

| Authentication Server Reply (AS_REP) 의 설명에서 보았던 것처럼, 티켓을 배분하기 전에 KDC 는 간단히 요청자와 서비스 제공자의 principal 이 데이터베이스에 존재하는지 확인합니다. TGT 에 대한 요청이었다면 더 쉽습니다. 왜냐하면 krbtgt/REALM@REALM 은 확실히 존재하기에, 단순한 초기 인증 요청으로 TGT 를 얻기 위해 사용자의 principal 이 존재하는지만 확인하는 것으로 충분합니다. 확실히 합법적이지 않은 사용자로부터 요청이 왔다면 TGT 는 사용될 수 없습니다. 그들은 암호를 모르기 때문에 유효한 인증자를 만들기 위한 세션 키를 얻을 수 없습니다. 그러나 쉽게 얻어진 이 티켓은 의도한 서비스의 장기간의 키를 추측하기 위한 시도로 무차별 대입 공격(brute-force attack) 을 겪을 수 있습니다. 분명 서비스의 비밀을 추측하는 것은 현재 연산처리 능력으로도 쉬운 일은 아닙니다. 하지만 Kerberos 5 에서 사전 인증(pre-authentication) 개념이 보안을 강화하기 위해 등장했습니다. 만일 KDC 의 정책이 (설정 가능) 초기 클라이언트 요청에 대해 사전 인증을 요청한다면, 인증 서버는 사전 인증이 필요하다는 에러 패킷으로 응답합니다. 클라이언트는 오류에 따라 사용자에게 암호 입력을 요청하고 요청을 재제출합니다. 그런데 이때 사용자의 장기 키로 암호화한 timestamp 를 더합니다. 장기 키는 알다시피 암호화되지 않은 비밀번호에 salt 가 있다면 더한 뒤 ``string2key`` 를 적용하여 얻어진 것입니다. 이번엔 KDC 는 사용자의 암호를 알기에 요청에 존재하는 timestamp 를 복호화 해봅니다. 이것이 성공하고 timestamp 가 기준 안에 들어왔다면, (설정된 허용오차 내), 요청한 사용자가 진짜고 인증 절차는 정상적으로 진행됩니다.
| 사전 인증은 KDC 정책이라는 점이고 프로토콜은 이걸 꼭 필요로 하는 것은 아니라는 점에 주목해야 합니다. 구현 관점에서 MIT Kerberos 5 와 Heimdal 은 기본적으로 사전인증이 비활성화 되어있습니다. 그러나 Windows Active Directory 와 AFS kaserver (pre-authenticated Kerberos 4) 의 Kerberos 는 사전 인증을 요청합니다.


---------------------------------
5. 티켓 심층 이해
---------------------------------

5.1 초기 티켓
===================

(사용자들은 비밀번호 입력으로 인증해야 할 때) 초기 티켓은 AS 로부터 직접 얻어 옵니다. 여기서 TGT 는 항상 초기 티켓이라는 점을 추론할 수 있습니다. 반면 서비스 티켓은 TGT 를 제출하는 것으로 TGS 로부터 배분됩니다. 따라서 초기 티켓이 아닙니다. 그러나 이 규칙에 예외가 있습니다.: 바로 몇초 전에 사용자가 비밀번호를 입력했다는 것을 보장하기 위해서 몇몇 Kerberos 애플리케이션은 서비스 티켓이 초기 티켓일 것을 요구합니다.; 이러한 경우에 티켓은 비록 TGT 가 아니지만 TGS 대신 AS 로부터 요청되고 초기 티켓이 됩니다. 운영 관점에서 pippo 라는 사용자가 mbox.example.com 장비에 있는 imap 서비스에 대한 초기 티켓을 얻고자 할 때 (즉, TGT 사용을 하지 않고) 다음 명령어를 사용합니다.:

::

    [pippo@client01 pippo]$ kinit -S imap/mbox.example.com@EXAMPLE.COM pippo@EXAMPLE.COM
    Password for pippo@EXAMPLE.COM:
    [pippo@client01 pippo]$
    [pippo@client01 pippo]$ klist -f
    Ticket cache: FILE:/tmp/krb5cc_500
    Default principal: pippo@EXAMPLE.COM

    Valid starting     Expires            Service principal
    01/27/05 14:28:59  01/28/05 14:28:39  imap/mbox.example.com@EXAMPLE.COM
            Flags: I

    Kerberos 4 ticket cache: /tmp/tkt500
    klist: You have no tickets cached

``I`` 플래그가 존재한다는 점에 유의합니다. 이는 초기 티켓을 나타냅니다.


5.2 갱신 가능한 티켓
======================

갱신가능한 티켓은 KDC 에 갱신하기 위하여 재제출될 수 있습니다.즉 전체 lifetime 이 재할당 됩니다. 명백히 KDC 는 티켓이 아직 만료되지 않았고 최대 갱신 시간을 (KDC 데이터베이스에 설정되어 있습니다.) 넘지 않았다면 갱신 요청을 받아들입니다. 티켓을 재갱신할 수 있는 것은 보안 상의 이유로 짧은 기간의 티켓을 가져야 한다는 필요성과 긴 시간동안 다시 비밀번호를 입력하지 않아야 하는 필요성을 모두 충족시킬 수 있습니다.: 예를 들면, 며칠 간 수행되어야 하고 사람의 추가 개입이 없어야 하는 작업을 상상해보십시오. 다음 예제는 pippo 가 최대 1시간 지속되지만 8일 간 갱신이 가능한 티켓을 요청하는 걸 보여줍니다.

::

    kinit -l 1h -r 8d pippo
    Password for pippo@EXAMPLE.COM:
    [pippo@client01 pippo]$
    [pippo@client01 pippo]$ klist -f
    Ticket cache: FILE:/tmp/krb5cc_500
    Default principal: pippo@EXAMPLE.COM

    Valid starting     Expires            Service principal
    01/27/05 15:35:14  01/27/05 16:34:54  krbtgt/EXAMPLE.COM@EXAMPLE.COM
            renew until 02/03/05 15:35:14, Flags: RI


    Kerberos 4 ticket cache: /tmp/tkt500
    klist: You have no tickets cached

다음은 pippo 가 비밀번호를 재입력하지 않고 티켓을 갱신하는 모습입니다.:

::

    [pippo@client01 pippo]$ kinit -R
    [pippo@client01 pippo]$
    [pippo@client01 pippo]$ klist -f
    Ticket cache: FILE:/tmp/krb5cc_500
    Default principal: pippo@EXAMPLE.COM

    Valid starting     Expires            Service principal
    01/27/05 15:47:52  01/27/05 16:47:32  krbtgt/EXAMPLE.COM@EXAMPLE.COM
            renew until 02/03/05 15:35:14, Flags: RIT


    Kerberos 4 ticket cache: /tmp/tkt500
    klist: You have no tickets cached


5.3 전달 가능한 티켓
==========================

| 다음 상황을 가정해 봅시다. 우리는 한 장비에서 관련된 TGT 와 함께 작업 세션을 갖고 있고 이 장비에서 다른 장비로 티켓을 유지하면서 로그인하고 싶습니다. 전달 가능한 티켓은 이 문제를 해결합니다. 한 장비에서 다른 장비로 전달된 티켓은 그자체로 전달 가능합니다. 따라서, 한번 인증되었다면 원하는 모든 장비로 아무 비밀번호도 재입력하지 않고 접속할 수 있습니다.
| Kerberos 없이 같은 결과를 얻고 싶다면 훨씬 덜 안전한 방법인 rsh 나 ssh 로 하는 공개키와 같은 방식을 사용해야 합니다. 그러나 후자의 방법은 사용자의 홈 디렉터리가 네트워크 파일시스템 (NFS 나 AFS) 에 있는 시스템에서는 실용적이지 않습니다. 비밀 키가(비밀이어야 하는) 네트워크로 전달되기 때문입니다.


-------------------------------
6. 교차 인증
-------------------------------

우리는 한 렐름에 속한 사용자가 인증하고 다른 렐름의 서비스에 접근하는 가능성을 언급한 적이 있습니다. 이 특징은 연관된 렐름들 간에 신뢰 관계가 있다는 가정에 기반을 둔 교차 인증이라고 부릅니다. 이는 A 렐름에 속한 사용자가 B 렐름의 서비스에 접근할 수 있고 그 반대 방향으로는 안되는 단방향일 수도 있고, 반대방향도 가능한 양방향일 수도 있습니다. 뒤이은 단락들에서 신뢰 관계를 직접적(direct), 전이적(transitive) 그리고 계층적(hierarchical) 으로 나누어 교차 인증을 살펴보겠습니다.


6.1 직접적 신뢰 관계
=============================

이 신뢰 관계 형식은 기본적이고 교차 인증의 기초가 되고 뒤에 살펴볼 다른 두 관계 형식을 구성하는데 쓰입니다. 이 관계는 B 렐름의 KDC 가 A 렐름의 KDC 를 직접 신뢰할 때 발생합니다. 따라서 A 렐름에 속한 사용자들이 B 렐름의 자원에 접근할 수 있도록 허용합니다. 실용적인 관점에서 직접 신뢰 관계는 두 연관된 KDC 가 키를 공유함으로써 얻어집니다. (양방향 신뢰가 필요하다면 키는 2개가 됩니다.) 이를 하기 위해서 원격 Ticket Granting Ticket 이 등장합니다. A 와 B 두개의 렐름에 대한 예제에서, ``krbtgt/B@A`` 형식이 같은 키로 양 KDC 에 추가됩니다. 이 키는 두 렐름 간에 신뢰를 보장하는 비밀입니다. 분명히 양방향으로 만들기 위해서는 (즉, A 도 B 를 신뢰한다면) 원격 TGT ``krbtgt/A@B`` 를 또 다른 비밀 키로 엮어 양 KDC 에 추가해야 합니다.

| 다음 예제에서 짧게 살펴볼텐데, 원격 TGT 들의 도입은 교차 인증을 일반 렐름 간 인증의 자연스럽게 일반화합니다.: 이는 Kerberos 의 동작에 대한 이전 설명이 한 렐름의 TGS 가 다른 렐름의 TGS 에 의해 발행된 원격 TGT 를 확인할 수 있는 한 계속 유효하다는 것을 강조합니다. 원격 TGT 가 AS 에 의해 발행되지 않을 때 발생하는 형식 이상에 주목합니다. 로컬 TGT 에서 발생하는 것처럼 말입니다. 그러나 로컬의 경우 로컬 TGT 를 로컬 Ticket Granting Server 에 제출했을 때 발생합니다.
| 이 모두를 명확하게 하기 위한 예제를 한번 살펴보겠습니다. EXAMPLE.COM 이라는 렐름의 pippo 라는 사용자가 있다고 가정합니다. 이 사용자의 principal 은 pippo@EXAMPLE.COM 입니다. 그리고 TEST.COM 렐름에 속한 pluto.test.com 서버에 ssh 를 통해 접속하고 싶어합니다.

- pippo 가 EXAMPLE.COM 렐름에서 TGT 를 아직 갖고 있지 않다면, 초기 인증 요청을(kinit) 합니다. 당연히 pippo 의 렐름에 있는 AS 로부터 응답이 옵니다.
- pippo 는 비밀번호 재입력 없이 pluto.test.com 에 원격 쉘을 열 수 있는 ``ssh pippo@pluto.test.com`` 명령을 내립니다.
- ssh 클라이언트는 2 개의 DNS 요청을 합니다.: pluto.test.com 의 IP 에 대한 요청 그리고 정규 형태로 호스트명을 (FQDN) 알아오기 위해 방금 알아온 IP 주소를 역으로 요청합니다.(이 예에서는 우연히 pluto.test.com 과 같습니다.)
- ssh 클라이언트는 이전 결과 덕분에 목적지가 사용자의 렐름에 속하지 않는다는 걸 깨닫고 EXAMPLE.COM 렐름의 TGS 에게 (이를 위해 자신의 렐름의 TGS 에게 요청한다는 점에 주목합니다.) 원격 TGT krbtgt/TEST.COM@EXAMPLE.COM 를 요청합니다.
- 원격 TGT 로 TEST.COM 렐름의 TGS 에게 host/pluto.test.com@TEST.COM 서비스 티켓을 요청합니다.
- TEST.COM Ticket Granting Service 가 요청을 받았을 때, krbtgt/TEST.COM@EXAMPLE.COM 이 자신의 데이터베이스에 있는지 확인합니다. 이게 있어야 신뢰 관계를 확인할 수 있습니다. 만일 확인이 성공하면 서비스 티켓(host/pluto.test.com@TEST.COM 의 키로 암호화된)은 마침내 발급되고 pippo 가 원격 쉘을 얻기 위해 pluto.test.com 호스트에 전달합니다.


6.2 전이적 신뢰 관계
===============================

교차 인증이 가능한 렐름의 수가 증가할 때, 교환해야 하는 키의 숫자는 이차적으로 증가합니다. 예를 들어, 5 개의 렐름이 있고 양방향의 관계여야 한다면 관리자는 20 개의 키를 만들어야 합니다. (double the combinations of 5 elements by 2 by 2).

이 문제를 해결하기 위해서 Kerberos 5 는 신뢰 관계에서 전이를 도입했습니다.: A 렐름이 B 렐름을 신뢰하고 B 렐름이 C 렐름을 신뢰한다면 A 도 자동적으로 C 를 신뢰합니다. 이 관계 특성은 키의 숫자를 눈에 띄게 줄입니다. (인증 통과 단계의 숫자가 늘어난다 하더라도).

그러나 여전히 문제가 있습니다.: 클라이언트는 직접적이지 않다면 인증 경로를 (capath) 추측할 수 없습니다. 따라서 클라이언트 각각의 설정에 특별한 절 ([capaths]) 을 만듦으로써 정확한 경로를 알아야 합니다. 이 경로는 KDC 에게도 알려져야 합니다. KDC 가 경로를 보고 환승 체계를 확인합니다.


6.3. 계층적 신뢰 관계
================================

만약 조직에서 대문자로 DNS 도메인 명으로 렐름 이름을 짓는 관습이 사용되고(가장 추천하는 방법), 후자가 계정에 속한다면, Kerberos 5 는 인접 렐름 (계층적으로) 이 신뢰 관계를 갖도록 지원하고 (자연스럽게 적절한 키의 존재로 가정된 신뢰가 지원되어야 합니다.) 자동적으로 전이 인증 경로를 (capaths 의 필요 없이) 구성합니다. 그러나 관리자들은 이 자동화된 방법을 클라이언트 설정에 capaths 를 강제하면서 변경할 수 있습니다. (효율성의 사유로)
