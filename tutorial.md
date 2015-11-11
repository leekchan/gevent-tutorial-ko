[TOC]

# 소개

이 튜토리얼은 중급 레벨의 파이썬 지식을 필요로 합니다. 동시성에 대한 선수 지식은 필요하지 않습니다. 이 튜토리얼의 목적은 독자에게 gevent를 사용하기 위한 도구들을 소개하고, 독자가 가지고 있었던 동시성 문제를 해결하고 비동기 어플리케이션을 작성하는것을 돕는것 입니다.

### Contributors

컨트리뷰터 리스트:
[Stephen Diehl](http://www.stephendiehl.com)
[J&eacute;r&eacute;my Bethmont](https://github.com/jerem)
[sww](https://github.com/sww)
[Bruno Bigras](https://github.com/brunoqc)
[David Ripton](https://github.com/dripton)
[Travis Cline](https://github.com/traviscline)
[Boris Feld](https://github.com/Lothiraldan)
[youngsterxyf](https://github.com/youngsterxyf)
[Eddie Hebert](https://github.com/ehebert)
[Alexis Metaireau](http://notmyidea.org)
[Daniel Velkov](https://github.com/djv)
[Sean Wang](https://github.com/sww)
[Inada Naoki](https://github.com/methane)
[Balthazar Rouberol](https://github.com/brouberol)
[Glen Baker](https://github.com/iepathos)
[Jan-Philip Gehrcke](https://gehrcke.de)
[Matthijs van der Vleuten](https://github.com/zr40)
[Simon Hayward](http://simonsblog.co.uk)
[Alexander James Phillips](https://github.com/AJamesPhillips)
[Ramiro Morales](https://github.com/ramiro)
[Philip Damra](https://github.com/djheru)
[Francisco Jos&eacute; Marques Vieira](https://github.com/fvieira)
[David Xia](https://www.davidxia.com)
[satoru](https://github.com/satoru)
[James Summerfield](https://github.com/jsummerfield)
[Adam Szkoda](https://github.com/adaszko)
[Roy Smith](https://github.com/roysmith)
[Jianbin Wei](https://github.com/jianbin-netskope)
[Anton Larkin](https://github.com/ToxicWar)
[Matias Herranz](https://github.com/matiasherranz-santex)
[Pietro Bertera](http://www.bertera.it)

gevent 튜토리얼을 쓰는데 도움을 주신 Denis Bilenko에게 감사의 뜻을 전합니다.

이 문서는 MIT license로 공개된 협업 문서입니다. 추가할 내용이나 오타를 발견하신 경우 [Github](https://github.com/sdiehl/gevent-tutorial)로 pull request를 보내주세요. 

한글 번역 오타 및 오역 수정은 [Github](https://github.com/leekchan/gevent-tutorial-ko)에 issue 생성 또는 pull request 부탁드립니다.

이 페이지는
[일본어](http://methane.github.com/gevent-tutorial-ja),
[중국어](http://xlambda.com/gevent-tutorial/),
[스페인어](http://ovnicraft.github.io/gevent/),
[이탈리아어](http://pbertera.github.io/gevent-tutorial-it/) and
[독일어](https://hellerve.github.io/gevent-tutorial-de)
로도 번역되어 있습니다.

# Core

## Greenlets

gevent에서 사용되는 주된 패턴은 <strong>Greenlet</strong>입니다. Greenlet은 C 확장 모듈 형태로 제공되는 경량 코루틴 입니다. Greenlet들은 메인 프로그램을 실행하는 OS process 안에서 모두 실행되지만 상호작용하며 스케줄링됩니다.

> 한 번에 오직 하나의 greenlet만이 실행됩니다.

운영체제에 의해 스케줄링되는 process들과 POSIX 쓰레드들을 사용하여 실제로 병렬로 실행되는 ``multiprocessing`` 나 ``threading``을 이용한 병렬처리들과는 다릅니다.


## Synchronous & Asynchronous Execution

동시성 처리의 핵심 개념은 큰 단위의 task를 한번에 *동기로* 처리하는 대신, 작은 단위의 subtask들로 쪼개서 동시에 *비동기*로 실행시키는 것입니다. 두 subtask간의 스위칭을 *컨텍스트 스위칭*이라고 합니다. 

gevent에서는 컨텍스트 스위칭을 *yielding*을 이용해서 합니다. 아래 코드는 ``gevent.sleep(0)``를 사용해 두 컨텍스트 사이에서 yield를 하는 예제입니다.

[[[cog
import gevent

def foo():
    print('Running in foo')
    gevent.sleep(0)
    print('Explicit context switch to foo again')

def bar():
    print('Explicit context to bar')
    gevent.sleep(0)
    print('Implicit context switch back to bar')

gevent.joinall([
    gevent.spawn(foo),
    gevent.spawn(bar),
])
]]]
[[[end]]]

위 예제는 컨텍스트 스위칭이 일어나는 모습과 실행 순서를 시각적으로 보여줍니다.

![Greenlet Control Flow](flow.gif)

gevent의 진짜 힘은 상호작용 하며 스케쥴링 될 수 있는 네트워크와 IO bound 함수들을 작성할때 발휘됩니다. gevent는 네트워크 라이브러리들이 암시적으로 greenlet 컨텍스트들이 가능한 시점에 암시적으로 yield 하도록 보장합니다. 이 개념은 정말 강조하고 싶은 중요한 개념인데, 예제를 통해서 살펴보도록 하겠습니다.

아래 코드는 ``select()`` 함수가 일반적인 blocking call을 하는 예시입니다.

[[[cog
import time
import gevent
from gevent import select

start = time.time()
tic = lambda: 'at %1.1f seconds' % (time.time() - start)

def gr1():
    # Busy waits for a second, but we don't want to stick around...
    print('Started Polling: %s' % tic())
    select.select([], [], [], 2)
    print('Ended Polling: %s' % tic())

def gr2():
    # Busy waits for a second, but we don't want to stick around...
    print('Started Polling: %s' % tic())
    select.select([], [], [], 2)
    print('Ended Polling: %s' % tic())

def gr3():
    print("Hey lets do some stuff while the greenlets poll, %s" % tic())
    gevent.sleep(1)

gevent.joinall([
    gevent.spawn(gr1),
    gevent.spawn(gr2),
    gevent.spawn(gr3),
])
]]]
[[[end]]]

아래 코드는 *non-deterministic*(i.e. 같은 입력에 대해 같은 결과를 보장하지 않음)한 ``task`` 함수를 정의하는 또 다른 인위적인 예제입니다. 이 함수 실행의 부작용은 task가 무작위 시간동안 실행되고 일시중지되는 것 입니다.

[[[cog
import gevent
import random

def task(pid):
    """
    Some non-deterministic task
    """
    gevent.sleep(random.randint(0,2)*0.001)
    print('Task %s done' % pid)

def synchronous():
    for i in range(1,10):
        task(i)

def asynchronous():
    threads = [gevent.spawn(task, i) for i in xrange(10)]
    gevent.joinall(threads)

print('Synchronous:')
synchronous()

print('Asynchronous:')
asynchronous()
]]]
[[[end]]]

동기(synchronous)처리시 모든 task들이 순차적으로 실행되고, 다른 task들이 각각 동작하는 동안 *blocking*(i.e. 프로그램의 동작을 일시중지함.)방식으로 동작합니다. 

이 프로그램에서 중요한 부분은 greentlet 쓰레드 안에서 주어진 함수를 감싸고 있는 ``gevent.spawn``입니다. 초기화된 greenlet들은 ``threads`` 배열안에 담겨서 ``gevent.joinall``로 넘겨집니다. 이때, ``gevent.joinall``은 모든 넘겨진 greenlet들이 실행될때 까지 block되어 있습니다. 이 greenlet들이 완전히 종료되고 나면 다음 코드가 실행됩니다.

주목해야할 중요한 사실은 비동기(asynchronous)처리시 실행 순서가 보장되지 않고 실행시간이 동기(synchronous)처리시보다 훨씬 줄어든다는 점입니다. 실제로 동기(synchronous)처리 예제를 완료하는데 걸리는 최대 시간은 각 task가 0.002초 동안 일시중지 될때 모두 실행되는데 0.02초입니다. 비동기(asynchronous)처리 예제에서는 task들이 서로 실행을 block하지 않으므로 최대 실행 시간이 약 0.002초입니다.

더 일반적인 예시로, 서버에서 비동기로 데이터들을 가져올때, ``fetch()`` 함수의 실행 시간이 서버의 부하상황에 따라 달라지는 경우가 있을 수 있습니다.

<pre><code class="python">import gevent.monkey
gevent.monkey.patch_socket()

import gevent
import urllib2
import simplejson as json

def fetch(pid):
    response = urllib2.urlopen('http://json-time.appspot.com/time.json')
    result = response.read()
    json_result = json.loads(result)
    datetime = json_result['datetime']

    print('Process %s: %s' % (pid, datetime))
    return json_result['datetime']

def synchronous():
    for i in range(1,10):
        fetch(i)

def asynchronous():
    threads = []
    for i in range(1,10):
        threads.append(gevent.spawn(fetch, i))
    gevent.joinall(threads)

print('Synchronous:')
synchronous()

print('Asynchronous:')
asynchronous()
</code>
</pre>

## Determinism

이전에 언급한것 처럼, greenlet은 deterministic 합니다. 같은 greenlet 세팅과 같은 입력이 주어졌을때, 언제나 같은 결과를 출력합니다. 예시로, task를 multiprocessing pool에서 나누어 실행시켜 보고 결과를 gevent pool에서 실행시켰을때와 비교해 보겠습니다.

<pre>
<code class="python">
import time

def echo(i):
    time.sleep(0.001)
    return i

# Non Deterministic Process Pool

from multiprocessing.pool import Pool

p = Pool(10)
run1 = [a for a in p.imap_unordered(echo, xrange(10))]
run2 = [a for a in p.imap_unordered(echo, xrange(10))]
run3 = [a for a in p.imap_unordered(echo, xrange(10))]
run4 = [a for a in p.imap_unordered(echo, xrange(10))]

print(run1 == run2 == run3 == run4)

# Deterministic Gevent Pool

from gevent.pool import Pool

p = Pool(10)
run1 = [a for a in p.imap_unordered(echo, xrange(10))]
run2 = [a for a in p.imap_unordered(echo, xrange(10))]
run3 = [a for a in p.imap_unordered(echo, xrange(10))]
run4 = [a for a in p.imap_unordered(echo, xrange(10))]

print(run1 == run2 == run3 == run4)
</code>
</pre>

<pre>
<code class="python">False
True</code>
</pre>

gevent가 일반적으로 deterministic 하다고 해도, 소켓과 파일같은 외부 서비스와 연동할때 non-deterministic한 입력들이 들어올 수 있습니다. 그러므로 green 쓰레드가 "deterministic
concurrency" 형태라고 해도, POSIX 쓰레드들과 프로세스들을 다룰때 만나는 문제들을 경험할 수 있습니다.

동시성을 다룰때 만날 수 있는 문제로 *race condition*이 있습니다. 간단히 요약하자면, race condition은 두 개의 동시에 실행되는 쓰레드나 프로세스들이 같은 공유 자원을 수정하려고 할때 발생합니다. 이 때 해당 공유자원의 결과 값은 실행 순서에 따라 달라지게 됩니다. 이런 결과는 non-deterministic한 프로그램 동작을 야기하기 때문에 발생시키지 않기 위해 노력해야 합니다. 

가장 좋은 접근법은 공유 자원을 사용하지 않는것 입니다. 공유자원을 잘못 사용하면 부작용이 엄청나니 주의해야 합니다!

## Spawning Greenlets

gevent는 greenlet 초기화를 위한 몇가지 wrapper들을 제공합니다. 
일반적인 패턴은 다음과 같습니다:

[[[cog
import gevent
from gevent import Greenlet

def foo(message, n):
    """
    Each thread will be passed the message, and n arguments
    in its initialization.
    """
    gevent.sleep(n)
    print(message)

# Initialize a new Greenlet instance running the named function
# foo
thread1 = Greenlet.spawn(foo, "Hello", 1)

# Wrapper for creating and running a new Greenlet from the named
# function foo, with the passed arguments
thread2 = gevent.spawn(foo, "I live!", 2)

# Lambda expressions
thread3 = gevent.spawn(lambda x: (x+1), 2)

threads = [thread1, thread2, thread3]

# Block until all threads complete.
gevent.joinall(threads)
]]]
[[[end]]]

Greenlet 클래스를 사용하는것 외에도, Greenlet 클래스를 상속하고 ``_run`` 함수를 override 하는 방법도 있습니다.

[[[cog
import gevent
from gevent import Greenlet

class MyGreenlet(Greenlet):

    def __init__(self, message, n):
        Greenlet.__init__(self)
        self.message = message
        self.n = n

    def _run(self):
        print(self.message)
        gevent.sleep(self.n)

g = MyGreenlet("Hi there!", 3)
g.start()
g.join()
]]]
[[[end]]]


## Greenlet State

다른 코드 예시들 처럼, greenlet도 다양한 경우에 실패할 수 있습니다. greenlet은 예외를 발생시키는것이 실패하거나, 정지에 실패할 수 도 있고, 시스템 자원을 과도하게 사용할 수 도 있습니다.

greenlet의 내부 상태는 대체로 time-dependent합니다. greenlet에는 쓰레드의 상태를 모니터링 할 수 있는 다양한 flag들이 있습니다:

- ``started`` -- Boolean, Greenlet이 실행되었는지 여부를 나타냅니다
- ``ready()`` -- Boolean, Greenlet이 정지되었는지 여부를 나타냅니다
- ``successful()`` -- Boolean, Greenlet이 예외를 발생시키지 않고 정지되었는지 여부를 나타냅니다.
- ``value`` -- 어떠한 타입도 가능, Greenlet에 의해서 리턴된 값입니다.
- ``exception`` -- exception, Greenlet안에서 발생한 예외입니다.

[[[cog
import gevent

def win():
    return 'You win!'

def fail():
    raise Exception('You fail at failing.')

winner = gevent.spawn(win)
loser = gevent.spawn(fail)

print(winner.started) # True
print(loser.started)  # True

# Exceptions raised in the Greenlet, stay inside the Greenlet.
try:
    gevent.joinall([winner, loser])
except Exception as e:
    print('This will never be reached')

print(winner.value) # 'You win!'
print(loser.value)  # None

print(winner.ready()) # True
print(loser.ready())  # True

print(winner.successful()) # True
print(loser.successful())  # False

# The exception raised in fail, will not propagate outside the
# greenlet. A stack trace will be printed to stdout but it
# will not unwind the stack of the parent.

print(loser.exception)

# It is possible though to raise the exception again outside
# raise loser.exception
# or with
# loser.get()
]]]
[[[end]]]

## Program Shutdown

메인 프로그램이 SIGQUIT 시그널을 받은 시점에 yield를 실패한 Greenlet은 예상보다 오래 실행이 정지되어 있을 수 있습니다. 이런 프로세스는 "좀비 프로세스"라고 불리고, 파이썬 인터프리터 외부에서 kill되어야 합니다.

일반적인 패턴은 메인 프로그램에서 SIGQUIT 시그널에 대기하고 있다가 프로그램이 종료되기 전에 ``gevent.shutdown`` 호출하는 것 입니다.

<pre>
<code class="python">import gevent
import signal

def run_forever():
    gevent.sleep(1000)

if __name__ == '__main__':
    gevent.signal(signal.SIGQUIT, gevent.kill)
    thread = gevent.spawn(run_forever)
    thread.join()
</code>
</pre>

## Timeouts

타임아웃은 코드 블럭이나 Greenlet의 실행시간에 제한을 주는 것 입닏.

<pre>
<code class="python">
import gevent
from gevent import Timeout

seconds = 10

timeout = Timeout(seconds)
timeout.start()

def wait():
    gevent.sleep(10)

try:
    gevent.spawn(wait).join()
except Timeout:
    print('Could not complete')

</code>
</pre>

아래와 같이 ``with`` statement를 사용해서 타임아웃을 줄 수도 있습니다.

<pre>
<code class="python">import gevent
from gevent import Timeout

time_to_wait = 5 # seconds

class TooLong(Exception):
    pass

with Timeout(time_to_wait, TooLong):
    gevent.sleep(10)
</code>
</pre>

또한, gevent는 Greenlet 호출과 관련된 타임아웃 파라미터들을 제공합니다. 예를 들면 다음과 같습니다:

[[[cog
import gevent
from gevent import Timeout

def wait():
    gevent.sleep(2)

timer = Timeout(1).start()
thread1 = gevent.spawn(wait)

try:
    thread1.join(timeout=timer)
except Timeout:
    print('Thread 1 timed out')

# --

timer = Timeout.start_new(1)
thread2 = gevent.spawn(wait)

try:
    thread2.get(timeout=timer)
except Timeout:
    print('Thread 2 timed out')

# --

try:
    gevent.with_timeout(1, wait)
except Timeout:
    print('Thread 3 timed out')

]]]
[[[end]]]

## Monkeypatching

유감스럽게도 gevent의 어두운 면을 보게 되었습니다. 지금까지 강력한 코루틴 패턴들을 사용해보기 위해 monkey patching에 대해 언급하지 않았었는데요, monkey patching의 어두운 면에 대해 언급할 때가 되었습니다. 위에서 ``monkey.patch_socket()``를 사용했던것을 눈치채셨는지 모르겠는데요, 이 명령은 파이썬 기본 소켓 라이브러리를 수정하여 부작용을 유발할 수 있는 명령입니다.

<pre>
<code class="python">import socket
print(socket.socket)

print("After monkey patch")
from gevent import monkey
monkey.patch_socket()
print(socket.socket)

import select
print(select.select)
monkey.patch_select()
print("After monkey patch")
print(select.select)
</code>
</pre>

<pre>
<code class="python">class 'socket.socket'
After monkey patch
class 'gevent.socket.socket'

built-in function select
After monkey patch
function select at 0x1924de8
</code>
</pre>

파이썬은 런타임에 모듈, 클래스, 심지어 함수까지도 수정하는것을 허용합니다. 이는 문제가 발생했을 때 디버깅을 매우 어렵게 만드는 "암시적인 부작용"을 유발할 수 있는 안좋은 생각입니다. 그럼에도 불구하고, monkey patching은 라이브러리가 파이썬의 기본 동작을 변경해야 하는 극단적인 상황에서 사용될 수 있습니다. monkey patching덕분에 gevent는 ``socket``, ``ssl``, ``threading``, 그리고 ``select`` 과 같은 기본 라이브러리들에 있는 blocking system call들이 동시에 실행될 수 있도록 수정할 수 있습니다.

예를 들어, Redis 파이썬 바인딩은 Redis 서버와 통신하기 위해 기본 tcp 소켓을 합니다. ``gevent.monkey.patch_all()``를 실행시키는 것 만으로 Redis 바인딩이 요청들을 동시에 실행될 수 있도록 스케쥴링 되도록 만들고 gevent 코드에서 동작하도록 만들 수 있습니다.

이는 별도의 코드 작성 없이도 라이브러리들을 연동하는 것을 가능하게 합니다. monkey patching은 여전히 악이지만, 이 경우에는 "필요악"입니다.

# Data Structures

## Events

Event는 Greenlet 간의 비동기 통신에 사용됩니다.

<pre>
<code class="python">import gevent
from gevent.event import Event

'''
Illustrates the use of events
'''


evt = Event()

def setter():
    '''After 3 seconds, wake all threads waiting on the value of evt'''
	print('A: Hey wait for me, I have to do something')
	gevent.sleep(3)
	print("Ok, I'm done")
	evt.set()


def waiter():
	'''After 3 seconds the get call will unblock'''
	print("I'll wait for you")
	evt.wait()  # blocking
	print("It's about time")

def main():
	gevent.joinall([
		gevent.spawn(setter),
		gevent.spawn(waiter),
		gevent.spawn(waiter),
		gevent.spawn(waiter),
		gevent.spawn(waiter),
		gevent.spawn(waiter)
	])

if __name__ == '__main__': main()

</code>
</pre>

Event 객체의 확장은 wakeup call과 함께 값을 전송할 수 있는 AsyncResult입니다. AsyncResult는 임의의 시간에 할당될 미래 값에 대한 레퍼런스를 갖고 있기 때문에, 때때로 future나 deferred로 불리기도 합니다.

<pre>
<code class="python">import gevent
from gevent.event import AsyncResult
a = AsyncResult()

def setter():
    """
    After 3 seconds set the result of a.
    """
    gevent.sleep(3)
    a.set('Hello!')

def waiter():
    """
    After 3 seconds the get call will unblock after the setter
    puts a value into the AsyncResult.
    """
    print(a.get())

gevent.joinall([
    gevent.spawn(setter),
    gevent.spawn(waiter),
])

</code>
</pre>

## Queues

Queue는 일반적인 ``put`` 과 ``get` 연산을 지원하지만 Greenlet 사이에서 안전하게 조작되는 것이 보장되는 순서를 가진 데이터들의 집합입니다.

예를 들어 한 Greenlet이 queue에서 값 하나를 가져왔을 때, 해당 값이 다른 Greenlet에 의해서 동시에 접근되지 못하도록 보장합니다.

[[[cog
import gevent
from gevent.queue import Queue

tasks = Queue()

def worker(n):
    while not tasks.empty():
        task = tasks.get()
        print('Worker %s got task %s' % (n, task))
        gevent.sleep(0)

    print('Quitting time!')

def boss():
    for i in xrange(1,25):
        tasks.put_nowait(i)

gevent.spawn(boss).join()

gevent.joinall([
    gevent.spawn(worker, 'steve'),
    gevent.spawn(worker, 'john'),
    gevent.spawn(worker, 'nancy'),
])
]]]
[[[end]]]

또한 Queue는 ``put``이나 ``get`` 연산시 block 됩니다. 

non-blocking 연산이 필요할 때는 block이 되지 않는 ``put_nowait``과 ``get_nowait`을 사용할 수 있습니다. 대신 연산이 불가능 할때는 ``gevent.queue.Empty`` 나 ``gevent.queue.Full`` 예외를 발생시킵니다.
Queues can also block on either ``put`` or ``get`` as the need arises.

아래 코드는 상사가 3명의 작업자(steve, john, nancy)에게 동시에 일을 시키는데 Queue가 3개 이상의 요소를 담지 않도록 제한하는 예시입니다. 이 제한은 ``put``연산이 Queue에 남은 공간이 있을때 까지 block 되어야 함을 의미합니다. 반대로 ``get`` 연산은 Queue에 요소가 없으면 block 되는데, 일정 시간이 지날 때 까지 요소가 들어오지 않으면 ``gevent.queue.Empty`` 예외를 발생시키면서 종료될 수 있도록 타임아웃 파라미터를 설정할 수 있습니다.

[[[cog
import gevent
from gevent.queue import Queue, Empty

tasks = Queue(maxsize=3)

def worker(name):
    try:
        while True:
            task = tasks.get(timeout=1) # decrements queue size by 1
            print('Worker %s got task %s' % (name, task))
            gevent.sleep(0)
    except Empty:
        print('Quitting time!')

def boss():
    """
    Boss will wait to hand out work until a individual worker is
    free since the maxsize of the task queue is 3.
    """

    for i in xrange(1,10):
        tasks.put(i)
    print('Assigned all work in iteration 1')

    for i in xrange(10,20):
        tasks.put(i)
    print('Assigned all work in iteration 2')

gevent.joinall([
    gevent.spawn(boss),
    gevent.spawn(worker, 'steve'),
    gevent.spawn(worker, 'john'),
    gevent.spawn(worker, 'bob'),
])
]]]
[[[end]]]

## Groups and Pools

Group은 동시에 관리되고 스케쥴링 되는 실행중인 Greenlet들의 집합입니다. 

[[[cog
import gevent
from gevent.pool import Group

def talk(msg):
    for i in xrange(3):
        print(msg)

g1 = gevent.spawn(talk, 'bar')
g2 = gevent.spawn(talk, 'foo')
g3 = gevent.spawn(talk, 'fizz')

group = Group()
group.add(g1)
group.add(g2)
group.join()

group.add(g3)
group.join()
]]]
[[[end]]]

Group은 비동기 task 집합들을 관리하는데 굉장히 유용합니다.

``Group``은 작업들을 Greenlet집합에서 실행시키고 결과들을 다양한 방법을 통해 수집할 수 있습니다.

[[[cog
import gevent
from gevent import getcurrent
from gevent.pool import Group

group = Group()

def hello_from(n):
    print('Size of group %s' % len(group))
    print('Hello from Greenlet %s' % id(getcurrent()))

group.map(hello_from, xrange(3))


def intensive(n):
    gevent.sleep(3 - n)
    return 'task', n

print('Ordered')

ogroup = Group()
for i in ogroup.imap(intensive, xrange(3)):
    print(i)

print('Unordered')

igroup = Group()
for i in igroup.imap_unordered(intensive, xrange(3)):
    print(i)

]]]
[[[end]]]

Pool은 동시에 제한된 개수의 Greenlet을 실행 시킬 수 있도록 해줍니다. Pool은 대량의 네트워크 또는 IO bound 작업들을 동시에 실행하는 경우에 유용합니다.

[[[cog
import gevent
from gevent.pool import Pool

pool = Pool(2)

def hello_from(n):
    print('Size of pool %s' % len(pool))

pool.map(hello_from, xrange(3))
]]]
[[[end]]]

종종 gevent 기반 서비스를 만들 때 전체 서비스를 Pool 기반으로 작성하게 됩니다. 아래 코드는 다양한 소켓들에 폴링하는 예시입니다.

<pre>
<code class="python">from gevent.pool import Pool

class SocketPool(object):

    def __init__(self):
        self.pool = Pool(1000)
        self.pool.start()

    def listen(self, socket):
        while True:
            socket.recv()

    def add_handler(self, socket):
        if self.pool.full():
            raise Exception("At maximum pool size")
        else:
            self.pool.spawn(self.listen, socket)

    def shutdown(self):
        self.pool.kill()

</code>
</pre>

## Locks and Semaphores

Semaphore는 Greenlet들이 동시에 접근하거나 실행되는것을 제한하는 저수준의 synchronization primitive입니다. Semaphore는 ``acquire``와 ``release``라는 함수를 가지고 있습니다. Semaphore가 acquire 되거나 release되는 숫자의 차이는 Semaphore bound라고 불립니다. Semaphore bound가 0에 도달하면 다른 Greenlet이 release 할 때 까지 block 됩니다.

[[[cog
from gevent import sleep
from gevent.pool import Pool
from gevent.coros import BoundedSemaphore

sem = BoundedSemaphore(2)

def worker1(n):
    sem.acquire()
    print('Worker %i acquired semaphore' % n)
    sleep(0)
    sem.release()
    print('Worker %i released semaphore' % n)

def worker2(n):
    with sem:
        print('Worker %i acquired semaphore' % n)
        sleep(0)
    print('Worker %i released semaphore' % n)

pool = Pool()
pool.map(worker1, xrange(0,2))
pool.map(worker2, xrange(3,6))
]]]
[[[end]]]

Semaphore bound가 1인 경우를 Lock이라고 합니다. Lock은 Greenlet이 하나만 실행되는 것을 보장합니다. Lock은 자원이 한번에 하나의 Greenlet에 의해서만 사용되는 것을 보장하여야 할 때 사용됩니다.

## Thread Locals

또한 Gevent는 Greenlet context 안에서 특정 변수를 지역 변수로 명시하는 것이 가능합니다. 이것은 내부적으로 Greenlet의 ``getcurrent()`` 값으로 private namespace key를 설정하는 방식으로 구현되어 있습니다.

[[[cog
import gevent
from gevent.local import local

stash = local()

def f1():
    stash.x = 1
    print(stash.x)

def f2():
    stash.y = 2
    print(stash.y)

    try:
        stash.x
    except AttributeError:
        print("x is not local to f2")

g1 = gevent.spawn(f1)
g2 = gevent.spawn(f2)

gevent.joinall([g1, g2])
]]]
[[[end]]]

gevent를 사용하는 많은 웹프레임워크들은 HTTP 세션 객체들을 gevent thread local 변수로 저장합니다. 예를 들어, Werkzeug 유틸리티 라이브러리와 해당 라이브러리의 proxy 객체를 사용하면 Flask 스타일의 request 객체를 만들 수 있습니다.

<pre>
<code class="python">from gevent.local import local
from werkzeug.local import LocalProxy
from werkzeug.wrappers import Request
from contextlib import contextmanager

from gevent.wsgi import WSGIServer

_requests = local()
request = LocalProxy(lambda: _requests.request)

@contextmanager
def sessionmanager(environ):
    _requests.request = Request(environ)
    yield
    _requests.request = None

def logic():
    return "Hello " + request.remote_addr

def application(environ, start_response):
    status = '200 OK'

    with sessionmanager(environ):
        body = logic()

    headers = [
        ('Content-Type', 'text/html')
    ]

    start_response(status, headers)
    return [body]

WSGIServer(('', 8000), application).serve_forever()


<code>
</pre>

Flask의 시스템은 이 예제보다 더 복잡하지만, thread local을 로컬 세션 저장소로 사용하는 개념은 유사합니다.

## Subprocess

gevent 1.0 부터, ``gevent.subprocess`` (파이썬의 ``subprocess`` 모듈의 패치된 버전)이 추가되었습니다. ``gevent.subprocess``는 subprocess들의 cooperative waiting을 지원합니다.

<pre>
<code class="python">
import gevent
from gevent.subprocess import Popen, PIPE

def cron():
    while True:
        print("cron")
        gevent.sleep(0.2)

g = gevent.spawn(cron)
sub = Popen(['sleep 1; uname'], stdout=PIPE, shell=True)
out, err = sub.communicate()
g.kill()
print(out.rstrip())
</pre>

<pre>
<code class="python">
cron
cron
cron
cron
cron
Linux
<code>
</pre>

또한 많은 사람들이  ``gevent``와 ``multiprocessing``를 동시에 사용하고 싶어합니다. ``multiprocessing``에서 제공되는 프로세스간 통신은 기본적으로 cooperative하지 않습니다. (예륻 들어, ``Pipe``와 같은) ``multiprocessing.Connection``를 기반으로 한 오브젝트들은 내부의 file descriptor를 노출하기 때문에, 실제로 reading/writing을 하기 전에 ready-to-read/ready-to-write 이벤트에 대기(cooperatively wait)하기 위해 ``gevent.socket.wait_read``와 ``wait_write``를 사용할 수 있습니다.

<pre>
<code class="python">
import gevent
from multiprocessing import Process, Pipe
from gevent.socket import wait_read, wait_write

# To Process
a, b = Pipe()

# From Process
c, d = Pipe()

def relay():
    for i in xrange(10):
        msg = b.recv()
        c.send(msg + " in " + str(i))

def put_msg():
    for i in xrange(10):
        wait_write(a.fileno())
        a.send('hi')

def get_msg():
    for i in xrange(10):
        wait_read(d.fileno())
        print(d.recv())

if __name__ == '__main__':
    proc = Process(target=relay)
    proc.start()

    g1 = gevent.spawn(get_msg)
    g2 = gevent.spawn(put_msg)
    gevent.joinall([g1, g2], timeout=1)
</code>
</pre>

하지만, ``multiprocessing``과 gevent를 함께 사용하면 OS와 관련된 문제들을 야기할 수 있습니다:

* POSIX호홤 시스템에서 [forking](http://linux.die.net/man/2/fork)후에는 자식 프로세스에서 gevent의 상태는 ill-posed상태입니다. 부작용은 ``multiprocessing.Process`` 생성 작업이 부모와 자식 프로세스에서 모두 되기 전에 Greenlet들이 생성되는것 입니다.
* ``put_msg()``함수 안에서 ``a.send()``를 호출하면 thread 호출을 block시킬 수 있습니다: ready-to-write 이벤트는 오직 1byte를 쓰는것만 보장합니다. 쓰기 시도가 완료되기 전에 이미 내부 버퍼가 가득찼을 수 도 있습니다.
* ``wait_write()`` / ``wait_read()`` 기반 접근은 윈도우에서는 동작하지 않습니다. (``IOError: 3 is not a socket (files are not supported)``) 윈도우는 pipe 이벤트를 감지할 수 없기 때문입니다.

파이썬 패키지 [gipc](http://pypi.python.org/pypi/gipc)은 POSIX 호환 시스템과 윈도우 시스템 양쪽에서 모두 투명하게 동작하는 시스템을 만들 수 있도록 해줍니다. gipc는 gevent와 호환되는 ``multiprocessing.Process``기반 자식 프로세스와 파이프에 기반한 gevent 호환 프로세스간 통신을 지원합니다.

## Actors

Actor 모델은 Erlang에 의해서 대중화된 고수준의 동시성 모델입니다. 핵심 개념을 요약하자면 서로 메시지를 주고받을 수 있는 독립된 Actor들의 집합을 사용할 수 있는것 입니다. Actor 안에 있는 메인 루프는 메시지들을 반복적으로 살펴보면서 해당 메시지의 명령들을 실행합니다.

Gevent는 primitive Actor 타입을 지원하지는 않지만, Greenlet을 상속한 클래스 안에서 Queue를 사용하여 간단하게 구현해볼 수 있습니다.

<pre>
<code class="python">import gevent
from gevent.queue import Queue


class Actor(gevent.Greenlet):

    def __init__(self):
        self.inbox = Queue()
        Greenlet.__init__(self)

    def receive(self, message):
        """
        Define in your subclass.
        """
        raise NotImplemented()

    def _run(self):
        self.running = True

        while self.running:
            message = self.inbox.get()
            self.receive(message)

</code>
</pre>

사용 예시:

<pre>
<code class="python">import gevent
from gevent.queue import Queue
from gevent import Greenlet

class Pinger(Actor):
    def receive(self, message):
        print(message)
        pong.inbox.put('ping')
        gevent.sleep(0)

class Ponger(Actor):
    def receive(self, message):
        print(message)
        ping.inbox.put('pong')
        gevent.sleep(0)

ping = Pinger()
pong = Ponger()

ping.start()
pong.start()

ping.inbox.put('start')
gevent.joinall([ping, pong])
</code>
</pre>

# Real World Applications

## Gevent ZeroMQ

[ZeroMQ](http://www.zeromq.org/)의 개발자는 ZeroMQ를 다음과 같이 정의합니다: "a socket library that acts as a concurrency framework". ZeroMQ는 동시성을 가진 분산시스템을 만들기 위한 강력한 messaging layer입니다.

ZeroMQ는 다양한 socket primitive들을 제공합니다. socket은 blocking 연산인 ``send``, ``recv` 함수를 제공합니다. 하지만 [Travis Cline](https://github.com/traviscline)가 만든 라이브러리를 사용하면 gevent.socket을 이용해 ZeroMQ socket에 non-blocking 방식으로 접근할 수 있습니다. 다음 명령어로 PyPi에서 gevent-zeromq를 설치할 수 있습니다: ``pip install
gevent-zeromq``

[[[cog
# Note: Remember to ``pip install pyzmq gevent_zeromq``
import gevent
from gevent_zeromq import zmq

# Global Context
context = zmq.Context()

def server():
    server_socket = context.socket(zmq.REQ)
    server_socket.bind("tcp://127.0.0.1:5000")

    for request in range(1,10):
        server_socket.send("Hello")
        print('Switched to Server for %s' % request)
        # Implicit context switch occurs here
        server_socket.recv()

def client():
    client_socket = context.socket(zmq.REP)
    client_socket.connect("tcp://127.0.0.1:5000")

    for request in range(1,10):

        client_socket.recv()
        print('Switched to Client for %s' % request)
        # Implicit context switch occurs here
        client_socket.send("World")

publisher = gevent.spawn(server)
client    = gevent.spawn(client)

gevent.joinall([publisher, client])

]]]
[[[end]]]

## Simple Servers

<pre>
<code class="python">
# On Unix: Access with ``$ nc 127.0.0.1 5000``
# On Window: Access with ``$ telnet 127.0.0.1 5000``

from gevent.server import StreamServer

def handle(socket, address):
    socket.send("Hello from a telnet!\n")
    for i in range(5):
        socket.send(str(i) + '\n')
    socket.close()

server = StreamServer(('127.0.0.1', 5000), handle)
server.serve_forever()
</code>
</pre>

## WSGI Servers

Gevent는 HTTP로 컨텐츠를 서빙하는 두 가지의 WSGI 서버(``wsgi``, ``pywsgi``)를 제공합니다. 

* gevent.wsgi.WSGIServer
* gevent.pywsgi.WSGIServer

gevent 1.0.x 버전 미만에서는, gevent는 libev대신 libevent를 사용했었습니다. libevent는 gevent의 ``wsgi`` 서버에서 사용되는 빠른 HTTP 서버를 포함하고 있습니다.

gevent 1.0.x 버전에서는 HTTP 서버가 내장되어 있지 않습니다. 대신에 ``gevent.wsgi``가 ``gevent.pywsgi``에 내장된 pure 파이썬 서버의 alias가 되었습니다. 

## Streaming Servers

**gevent 1.0.x를 사용한다면, 이 섹션은 해당되지 않습니다**

streaming HTTP 서비스에 대해서 설명하자면, 핵심 개념은 헤더에서 컨텐츠의 길이를 명시하지 않는것 입니다. 대신에 커넥션을 열어둔 상태에서 파이프에 담긴 chunk들을 flush 합니다. 이 때 각 chunk의 길이를 알려주는 hex digit을 chunk앞에 붙입니다. 스트림은 길이가 0인 chunk가 전달되면 종료됩니다.

    HTTP/1.1 200 OK
    Content-Type: text/plain
    Transfer-Encoding: chunked

    8
    <p>Hello

    9
    World</p>

    0

wsgi는 streaming을 지원하지 않기 때문에 위 HTTP 커넥션은 wsgi에서 구현할 수 없습니다. 대신에 버퍼를 사용하면 가능합니다.

<pre>
<code class="python">from gevent.wsgi import WSGIServer

def application(environ, start_response):
    status = '200 OK'
    body = '&lt;p&gt;Hello World&lt;/p&gt;'

    headers = [
        ('Content-Type', 'text/html')
    ]

    start_response(status, headers)
    return [body]

WSGIServer(('', 8000), application).serve_forever()

</code>
</pre>

pywsgi를 사용하여 handler를 generator로 작성하고 응답을 chunk 단위로 yield할 수 있습니다.

<pre>
<code class="python">from gevent.pywsgi import WSGIServer

def application(environ, start_response):
    status = '200 OK'

    headers = [
        ('Content-Type', 'text/html')
    ]

    start_response(status, headers)
    yield "&lt;p&gt;Hello"
    yield "World&lt;/p&gt;"

WSGIServer(('', 8000), application).serve_forever()

</code>
</pre>

그럼에도 불구하고, gevent 서버의 성능은 다른 파이썬 서버들에 비해서 매우 빠릅니다. libev는 검증된 기술이고 libev로 구현된 서버들은 대규모 환경에서 잘 동작한다고 알려져 있습니다.

다른 서버들과 비교해 보려면 Apache 벤치마크툴인 ``ab``를 이용하거나 [Benchmark of Python WSGI Servers](http://nichol.as/benchmark-of-python-web-servers)를 보시면 됩니다.

<pre>
<code class="shell">$ ab -n 10000 -c 100 http://127.0.0.1:8000/
</code>
</pre>

## Long Polling

<pre>
<code class="python">import gevent
from gevent.queue import Queue, Empty
from gevent.pywsgi import WSGIServer
import simplejson as json

data_source = Queue()

def producer():
    while True:
        data_source.put_nowait('Hello World')
        gevent.sleep(1)

def ajax_endpoint(environ, start_response):
    status = '200 OK'
    headers = [
        ('Content-Type', 'application/json')
    ]

    start_response(status, headers)

    while True:
        try:
            datum = data_source.get(timeout=5)
            yield json.dumps(datum) + '\n'
        except Empty:
            pass


gevent.spawn(producer)

WSGIServer(('', 8000), ajax_endpoint).serve_forever()

</code>
</pre>

## Websockets

Websocket 예제는 <a href="https://bitbucket.org/Jeffrey/gevent-websocket/src">gevent-websocket</a>를 필요로 합니다.

<pre>
<code class="python"># Simple gevent-websocket server
import json
import random

from gevent import pywsgi, sleep
from geventwebsocket.handler import WebSocketHandler

class WebSocketApp(object):
    '''Send random data to the websocket'''

    def __call__(self, environ, start_response):
        ws = environ['wsgi.websocket']
        x = 0
        while True:
            data = json.dumps({'x': x, 'y': random.randint(1, 5)})
            ws.send(data)
            x += 1
            sleep(0.5)

server = pywsgi.WSGIServer(("", 10000), WebSocketApp(),
    handler_class=WebSocketHandler)
server.serve_forever()
</code>
</pre>

HTML Page:

    <html>
        <head>
            <title>Minimal websocket application</title>
            <script type="text/javascript" src="jquery.min.js"></script>
            <script type="text/javascript">
            $(function() {
                // Open up a connection to our server
                var ws = new WebSocket("ws://localhost:10000/");

                // What do we do when we get a message?
                ws.onmessage = function(evt) {
                    $("#placeholder").append('<p>' + evt.data + '</p>')
                }
                // Just update our conn_status field with the connection status
                ws.onopen = function(evt) {
                    $('#conn_status').html('<b>Connected</b>');
                }
                ws.onerror = function(evt) {
                    $('#conn_status').html('<b>Error</b>');
                }
                ws.onclose = function(evt) {
                    $('#conn_status').html('<b>Closed</b>');
                }
            });
        </script>
        </head>
        <body>
            <h1>WebSocket Example</h1>
            <div id="conn_status">Not Connected</div>
            <div id="placeholder" style="width:600px;height:300px;"></div>
        </body>
    </html>


## Chat Server

마지막 예제는, 실시간 채팅방 입니다. 이 예제는 <a href="http://flask.pocoo.org/">Flask</a>를 필요로 합니다 (Django, Pyramid 등 다른 프레임워크를 사용해도 됩니다). 관련된 Javascript와 HTML 파일들은 <a href="https://github.com/sdiehl/minichat">을 참조하세요.


<pre>
<code class="python"># Micro gevent chatroom.
# ----------------------

from flask import Flask, render_template, request

from gevent import queue
from gevent.pywsgi import WSGIServer

import simplejson as json

app = Flask(__name__)
app.debug = True

rooms = {
    'topic1': Room(),
    'topic2': Room(),
}

users = {}

class Room(object):

    def __init__(self):
        self.users = set()
        self.messages = []

    def backlog(self, size=25):
        return self.messages[-size:]

    def subscribe(self, user):
        self.users.add(user)

    def add(self, message):
        for user in self.users:
            print(user)
            user.queue.put_nowait(message)
        self.messages.append(message)

class User(object):

    def __init__(self):
        self.queue = queue.Queue()

@app.route('/')
def choose_name():
    return render_template('choose.html')

@app.route('/&lt;uid&gt;')
def main(uid):
    return render_template('main.html',
        uid=uid,
        rooms=rooms.keys()
    )

@app.route('/&lt;room&gt;/&lt;uid&gt;')
def join(room, uid):
    user = users.get(uid, None)

    if not user:
        users[uid] = user = User()

    active_room = rooms[room]
    active_room.subscribe(user)
    print('subscribe %s %s' % (active_room, user))

    messages = active_room.backlog()

    return render_template('room.html',
        room=room, uid=uid, messages=messages)

@app.route("/put/&lt;room&gt;/&lt;uid&gt;", methods=["POST"])
def put(room, uid):
    user = users[uid]
    room = rooms[room]

    message = request.form['message']
    room.add(':'.join([uid, message]))

    return ''

@app.route("/poll/&lt;uid&gt;", methods=["POST"])
def poll(uid):
    try:
        msg = users[uid].queue.get(timeout=10)
    except queue.Empty:
        msg = []
    return json.dumps(msg)

if __name__ == "__main__":
    http = WSGIServer(('', 5000), app)
    http.serve_forever()
</code>
</pre>
