========
Tutorial
========

이 튜토리얼은 **Tumblelog**블로그 어플리케이션을 만들며 **Mongoengine**을 설명합니다.
Tumblelog은 텍스트, 이미지, 링크, 영상, 오디오 등등의 미디어 컨텐츠를 지원하는 블로그 어플리케이션 입니다.
튜토리얼을 간단하게 하기 위해 우리의 Tumblelog은 텍스트, 이미지, 링크만 지원합니다.
그리고 이 튜토리얼의 목적은 MongoEngine을 설명하는 것이기 때문에 유저인터페이스 보다는
데이터 모델링에 중점을 두고 설명할 것입니다.

Getting started
===============

시작하기 전에 MongoDB가 접근 가능한 위치에서 실행되고 있어야 합니다. --- 만약 로컬에서 실행되고 있다면 훨씬 수월할 것입니다.
그렇지 않다면 리모트서버에서 실행이 되고 있어야 합니다.
만약 당신의 컴퓨터에 MongoEngien이 설치가 되어있지 않다면 pip를 이용하여 다음과 같이 쉽게 설치할 수 있습니다. ::

    $ pip install mongoengine

MongoEngine을 사용하기 전에  :func:`~mongoengine.connect` 함수를 사용하여
:program:`mongod`의 인스턴스를 연결하는 법을 알려드리겠습니다.
만약에 로컬에서 실행되고 있다면 함수의 매개변수로 MongoDB의 이름만 들어가면 됩니다.::

    from mongoengine import *

    connect('tumblelog')

MongoDB를 연결할 때 수많은 옵션을 사용할 수 있습니다.
더 자세한 정보는 :ref:`guide-connecting`가이드를 확인하세요.

Defining our documents
======================

MongoDB는 스키마를 정의할 필요가 없는 *스키마리스* 데이터베이스입니다. ---
필드를 추가하거나 지워도 에러가 발생하지 않습니다. 이는 데이터 모델 변경 등등 많은 것들을 편하게 해주지만
도큐먼트의 스키마를 정의하는 것은 잘못된 타입 또는 빠진 필드 등의 버그를 잡는 데에 도움을 줍니다.
그리고 :abbr:`ORMs (Object-Relational Mappers)`처럼 유틸리티 메소드를 정의할 수 있습니다.

Tumblelog 어플리케이션에서는 다양한 타입의 정보들을 저장해야 합니다.
먼저 post들을 각각의 유저마다 연결하기 위해 **users** 컬렉션이 필요합니다.
또한, 다양한 종류의 **posts**(예: 텍스트, 이미지, 링크)를 데이터베이스에 저장해야 합니다.
Tumblelog에서 post마다 **Tags**를 달면 특정 tag를 검색했을 때 글을 찾기 쉽게 해줍니다.
마지막으로 **comments**를 추가하면 멋진 Tumblelog가 만들어집니다.
다른 많은 도큐먼트들과 연결되어 있는 **users**를 먼저 정의해봅시다.

Users
-----

관계형 데이터베이스에서 ORM을 쓰는 것 처럼 :class:`User` 가 어떠한 필드를 가지고 있는지,
어떠한 타입의 데이터가 들어갈 것인지를 정의해야 합니다.::

    class User(Document):
        email = StringField(required=True)
        first_name = StringField(max_length=50)
        last_name = StringField(max_length=50)

위 코드는 일반적인 ORM에서 table의 구조를 정의하는 것과 비슷해 보입니다.
하지만 가장 중요한 차이점은 스키마가 MongoDB에 전달되거나 정의되지 않는다는 것입니다. ---
이것은 추후 수정 사항을 쉽게 관리하기 위해 어플리케이션 레벨에서 적용이 됩니다.
User 도큐먼트는 MongoDB의 table이 아닌 *collection*에 저장이 될 것입니다.

Posts, Comments and Tags
------------------------

이제 남은 정보들을 어떻게 저장할 것인지 생각해봅시다. 만약 관계형 데이터베이스를 사용했다면
아마 **posts** table과 **comments** table 그리고 **tags** table을 만들었을 것입니다.
그리고 comments와 각각의 posts를 연결하기 위해 comments table의 속성에 posts table의
foreign key를 넣었을 것입니다. 그리고 post table과 tag table을 연결하기 위해 many-to-many
relationship을 제공했을 것입니다. 그다음에 각각의 데이터 타입들(텍스트, 이미지, 링크)을 저장하는
문제를 해결해야 합니다. 이 문제를 해결하는 데에는 다양한 방법이 있지만, 각각의 방법마다 단점이 있어서
완벽한 해결책은 없습니다.

Posts
^^^^^

행복하게도 MongoDB는 관계형 데이터베이스가 *아니기* 때문에 위에 말했던 방법을 사용하지 않습니다.
MongoDB의 schemaless 환경이 더 멋진 해결책을 제공해주기 때문입니다.
우리는 모든 종류의 posts를 *한 collection*에 저장하고 각각의 종류의 posts를 위한 필드를 정의합니다.
만약에 나중에 비디오 posts를 추가하고 싶더라도 collection의 전체 내용을 수정할 필요가 없습니다.
단지 비디오 posts를 지원할 새로운 필드를 만들기만 하면 됩니다.
이것은 객체지향의 상속 원칙에 멋지게 알맞습니다. 우리는 :class:`Post`를 부모 클래스.
:class:`TextPost`, :class:`ImagePost` 그리고 :class:`LinkPost`를 :class:`Post`의
자식 클래스로 생각하면 됩니다. 사실 MongoEngine은 이 것을 창조적은 모델링의 한 종류로 지원합니다. ---
상속이 가능하게 설정하기 위해서는 :attr:`meta` 안에 있는 :attr:`allow_inheritance`를 True로
설정하기만 하면 됩니다.::

    class Post(Document):
        title = StringField(max_length=120, required=True)
        author = ReferenceField(User)

        meta = {'allow_inheritance': True}

    class TextPost(Post):
        content = StringField()

    class ImagePost(Post):
        image_path = StringField()

    class LinkPost(Post):
        link_url = StringField()

:class:`~mongoengine.fields.ReferenceField`를 이용하여 게시글 작성자의 Reference를 저장했습니다.
이는 전통적인 ORM의 foreign key 필드와 비슷합니다. 그리고 데이터를 저장할 때 자동으로 참조가 되고
데이터를 불러올 때 역참조가 됩니다.

Tags
^^^^

이제 Post 모델을 이해했으니 여기에 tags를 어떻게 붙일 수 있을까요?
MongoDB는 자체적으로 아이템들의 리스트를 저장할 수 있습니다. 각각 포스트의 태그들의
리스트를 Post 모델에 저장할 수 있습니다. 효율성과 간편성을 위해서 태그들을 collection에 reference로
분할하여 저장하지 않고 post에 string으로 저장하겠습니다. 특별히 tags는 매우 짧기 때문에(주로
도큐먼트의 id보다도 짧습니다.) 제한하지 않아도 데이터베이스의 크기에 크게 영향을 주지 않습니다.
수정된 :class:`Post` 클래스를 확인해봅시다.::

    class Post(Document):
        title = StringField(max_length=120, required=True)
        author = ReferenceField(User)
        tags = ListField(StringField(max_length=30))

Post의 tags를 저장하기 위해 사용한 :class:`~mongoengine.fields.ListField` 객체는 첫 번째
인자로 필드 객체를 받습니다. --- 이것은 어떤 종류의 필드의 리스트도 저장할 수 있다는 것입니다.
(리스트를 포함해서 말입니다.)

.. note::
    :class:`Post`를 상속받았기 때문에 각각의 post 종류마다 다 수정해줄 필요가 없습니다.

Comments
^^^^^^^^

comment는 전형적으로 *한* post에 연결이 되어 있습니다. 관계형 데이터베이스에서는 post와
그 post의 comments를 보여주기 위해 데이터베이스에서 post를 검색한 후 해당 post의 댓글을
다시 query를 했습니다. 관계형 데이터베이스를 사용하지 않는 이상 굳이 comments를 연결되어
있는 posts와 따로 저장할 이유가 없습니다. MongoDB를 사용하면 comments를 post document에
*embedded documents*의 리스트로 직접 저장할 수 있습니다. embedded documet는 다른 평범한
document와 다른 취급을 받지 않습니다. 단지 데이터베이스에서 자신만의 collection을 가지지
못할 뿐입니다. MngoEngine에서 embedded document의 구조를 평범한 document처럼 utility
메소드를 사용하여 정의할 수 있습니다.::

    class Comment(EmbeddedDocument):
        content = StringField()
        name = StringField(max_length=120)

post document에서 comment documents의 리스트를 저장할 수 있습니다.::

    class Post(Document):
        title = StringField(max_length=120, required=True)
        author = ReferenceField(User)
        tags = ListField(StringField(max_length=30))
        comments = ListField(EmbeddedDocumentField(Comment))

Handling deletions of references
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

:class:`~mongoengine.fields.ReferenceField` 객체는 참조되는 객체가 지워질 때의
삭제 규칙을 제어하기 위해 `reverse_delete_rule` 키워드를 가집니다. user가 지워질 때
모든 posts를 삭제하기 위해서 규칙을 설정해야 합니다.::

    class Post(Document):
        title = StringField(max_length=120, required=True)
        author = ReferenceField(User, reverse_delete_rule=CASCADE)
        tags = ListField(StringField(max_length=30))
        comments = ListField(EmbeddedDocumentField(Comment))

더 자세한 정보는 :class:`~mongoengine.fields.ReferenceField`를 확인하세요.

.. note::
    현재 Map필드와 Dict필드는 삭제된 참조에 대한 제어 방법이 지원되지 않습니다.

Adding data to our Tumblelog
============================

지금까지 document가 어떤 구조로 저장될 것인지 정의했습니다. 그러면 데이터베이스에
몇 개의 documents를 추가해 봅시다! 먼저 :class:`User` 객체를 만들어 봅시다.::

    ross = User(email='ross@example.com', first_name='Ross', last_name='Lawley').save()

.. note::
    user를 속성을 통해서 정의할 수도 있습니다.::

        ross = User(email='ross@example.com')
        ross.first_name = 'Ross'
        ross.last_name = 'Lawley'
        ross.save()

위에서 ``ross`` user를 만든 것처럼 ``john``이라는 user도 만들어 봅시다.

users를 데이터베이스에 넣었으니 두 개의 post를 추가해봅시다.::

    post1 = TextPost(title='Fun with MongoEngine', author=john)
    post1.content = 'Took a look at MongoEngine today, looks pretty cool.'
    post1.tags = ['mongodb', 'mongoengine']
    post1.save()

    post2 = LinkPost(title='MongoEngine Documentation', author=ross)
    post2.link_url = 'http://docs.mongoengine.com/'
    post2.tags = ['mongoengine']
    post2.save()

.. note::
    이미 :meth:`save` 메소드를 사용하여 저장된 객체에서 필드를 수정했더라도
    :meth:`save` 메소드를 다시 호출하면 새로운 내용으로 업데이트 됩니다.

Accessing our data
==================

데이터베이스에 저장된 두 개의 posts를 어떻게 보여줄 수 있을까요? 각각의
document class (i.e. :class:`~mongoengine.Document`를 직접적이든 간접적이든 상속받은 class)는
해당 class와 연관된 데이터베이스의 collection 내에 있는 documents에 접근할 때 사용되는
:attr:`objects` 속성을 가지고 있습니다. 그럼 우리 posts의 제목을 가져와봅시다.::

    for post in Post.objects:
        print(post.title)

Retrieving type-specific information
------------------------------------

이것은 posts의 제목을 한 줄에 하나씩 출력할 것입니다. 하지만 만약 TextPost의 특정 종류의 데이터
(link_url, content, etc.)를 접근하고자 하면 어떻게 해야 할까요? 간단하게 :class:`Post`의
자식 클래스의 :attr:`objects` 속성을 이용하는 방법이 있습니다.::

    for post in TextPost.objects:
        print(post.content)

TextPost의 :attr:`objects` 속성을 사용하면 :class:`TextPost`를 사용하여 생성된 documents들만
반환됩니다. 여기에는 일반적인 규칙이 있습니다.: :class:`~mongoengine.Document`의 서브 클래스의
:attr:`objects` 속성은 해당 서브 클래스 또는 해당 서브 클래스의 서브 클래스들을 통해 생성된 documents만
반환합니다.

그러면 특정 타입에 부합하는 posts만을 어떻게 보여줄 수 있을까요? 각각의 서브 클래스마다 :attr:`objects`
속성을 사용하는 것보다 더 좋은 방법이 있습니다. :class:`Post`에서 :attr:`objects` 속성을 사용하면
반환되는 값들은 :class:`Post`의 인스턴스가 아니라 :class:`Post`의 서브 클래스의 인스턴스입니다.
실제로 어떻게 작동하는지 아래 코드를 봅시다.::

    for post in Post.objects:
        print(post.title)
        print('=' * len(post.title))

        if isinstance(post, TextPost):
            print(post.content)

        if isinstance(post, LinkPost):
            print('Link: {}'.format(post.link_url))

위 코드는 각각의 posts 제목을 출력하고 텍스트 post라면 post의 content를 출력하고
링크 post라면 "Link: <url>"를 출력하게 됩니다.

Searching our posts by tag
--------------------------

:class:`~mongoengine.Document` class의 :attr:`objects` 속성은
:class:`~mongoengine.queryset.QuerySet` 객체입니다. 이것은 데이터를 요구할 때마다
데이터베이스에 게으른 query를 합니다. 또한, 필터링하여 반환되는 값들을 줄일 수도 있습니다.
그럼 tag가 "mongodb"인 posts만 반환이 되도록 query를 바로잡아봅시다.::

    for post in Post.objects(tags='mongodb'):
        print(post.title)

:class:`~mongoengine.queryset.QuerySet`에 있는 메소드들은 반환되는 결과를 다르게 할 수 있습니다.
예를 들어, :attr:`objects` 속성에 있는 :meth:`first` 메소드는 query에 첫 번째로 매치되는
하나의 document를 반환합니다. 집합에 관련된 함수 또한 :class:`~mongoengine.queryset.QuerySet`
객체에서 자주 사용됩니다.::

    num_posts = Post.objects(tags='mongodb').count()
    print('Found {} posts with tag "mongodb"'.format(num_posts))

Learning more about MongoEngine
-------------------------------

만약 이 튜토리얼을 완료했다면 당신은 매우 훌륭한 출발을 했습니다! 잘하셨습니다!!
MongoEngine을 배우는 여정의 다음 발자국은 MongoEngine과 MongoDB를 심도 있게 사용하는 법을
배울 수 있는 `full user guide <guide/index.html>`_ 입니다.
