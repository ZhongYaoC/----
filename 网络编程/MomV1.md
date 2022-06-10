### FdOBJ �׽��ֺ�����װ

#### ������飺

1. �о���getsockopt���ص�error�ĺ���
2. getsockopt �ķ���ֵ����
3. getsockopt �� WSAGetLastError���߷��ص�errorֵ������
4. ��ipʱ��ip��Ӧ��λ���� sockaddr_in.sin_addr.s_addr
5. win32��linux���������ƺ�����������Ӧ��
6. linger��NoDelay�ĺ��弰Ӧ��



```#include <stdio.h>``` 

stdio�⺯��Ϊ����ʹ��ϵͳ���ã����Ի�ʹ���ڲ��Ļ�����stdio����������fgets�����ͻ��õ��û�������
���ֺ����ڲ��Ļ�������ϵͳ����ʱ�ǿ������ģ�����select��ϵͳ���ú����޷�֪�����໺�������Ƿ���ֵ���ڣ�

### Mutex ��
#### ������
1. ������ spinlock
  ��ѯ�����޵ȴ���ֻҪ��ǰ��Դ�������߳�ռ�ã����߳̾Ͳ���ѯ���Ƿ��ռ��
  ���ȵȴ����˷�CPU��Դ
  �����ڣ�������ҵ��������кܿ�ĳ������ɱ��⻥�����µ��߳��л�

2. ������/�� mutex ��mutual exclusive��
  ������ѯ�����޵ȴ�����һ��������ȥ��������������ǰ��Դռ�ñ��ͷź�֪ͨ�̣߳������߳�˯��
  Ϊ�˹�������������ҪOS������ȣ��ʻ�������ʽ���Ӻ���Դ���漰�������л�

�����÷���
����һ���̼߳乲��Ļ�����mutex�����߳�ȥ����mutex��˭����˭ʹ����Դ

����mutex�� ��������ռ����Դ����


3. �������� condition variable
  ��ֹҵ���߼��г������޵ȴ� �������(����ʹ���˻���������Ȼ�������������е�����ѭ������)���Ӷ��˷�CPU��Դ����ҵ������������ϵͳ�޷�������������������
  ҵ������δ���ǰ���߳�˯�ߣ����������notify�̣߳���������

�������������̼߳�ͨѶ���ƣ��뻥����һ��ʹ�ã�

> ���������Ѿ��������ĸ����ˣ������ڻ�ȡ����Դ����Ȩ�޺�����ʵ��ҵ�񳡾��ֲ��ܷ��ʵ������Ҫ���ʵ��ִ�и�Ч

4. ��д�� readers-writers lock
  ��OS�ź����еĶ�д�����⣬д�߻��⣬���߼䲻���⣨������д��ռ����
  ������spinlock����mutexʵ��

**MR Hu���飺ֱ���û����������ö�д�������߿��ܻ���߼�����**
```C++
/*����PV�����ж�д������
* �����ȣ�д�߿��ܼ���
*
*��spinlock����mutexʵ�ֶ�д��
*/
//������������
������1 = 1;
//������д����
������2 = 1;

�������� = 0;
���߼���()
{
    ����-������1;
    ��������++;
    if ���������� == 1��
    {
        //��һ�����ߵ��������д�߽��룬����
        ����-������2;
    }
    ����-������1;
}

���߽���()
{
    ����-������1;
    ��������--;
    if (�������� == 0)
    {
        ����-������2;
    }
    ����-������1;
}

д�߼���()
{
    ����-������2;
}

д�߽���()
{
    ����-������2;
}
```

#### pthread API
##### ������
```C++
//����һ��������
pthread_mutex_t mtx;
//��ʼ��
pthread_mutex_init(&mtx, NULL);
//����
pthread_mutex_lock(&mtx);
//����
pthread_mutexx_unlock(&mtx);
//����
pthread_mutex_destroy(&mtx);

```

����������˯�ߵȴ����ͣ����Ժ�IOģ�����ƣ�����������������汾���������߳��޷���ȡ������ʱ�����޷���ȡ��Դʱ������˯�ߣ�������������һ������Ĵ���ֵ����```trylock```
```C++
ret = pthread_mutex_trylock(&mtx);
//�����ɹ�
if (ret == 0)
{
//do something...
pthread_mutex_unlock(&mtx);
}
//�����ڱ�ʹ��
else if (ret == EBUSY)
{
//some address...
}
```

����������ͬһ�߳��ܷ��μ��������������ռ����Դ������:
��Ϊ�ݹ黥����recursive mutex�������������ͷǵݹ黥����non-recursive mutex��������������

ͬһ�̶߳Էǵݹ黥������μ�������������������ݹ黥�����޴˷���
pthread��ͨ����mutex���```PTHREAD_MUTEX_RECURSIVE```���Եķ�ʽ��ʹ�õݹ黥����
```C++
//����һ��������
pthread_mutex_t mtx;
//����һ�������������Ա���
pthread_mutexattr_t mtx_attr;

//��ʼ�������������Ա���
pthread_mutexattr_init(&mtx_attr);
//���û�����������Ϊ�ݹ�
pthread_mutexattr_settype(&mtx_attr, PTHREAD_MUTEX_RECURSIVE);

//���Ը�ֵ��������
pthread_mutex_init(&mtx, &mtx_attr);
```
>ʹ�úõݹ黥������ʮ��tricky�ģ�����û�������������ʱ��ʹ��


##### ��д��
��д����ʵ��һ�������������Ϊһ�����Ķ�ģʽ��дģʽ��

���ԣ�
����д��������д��ʱ�������̶߳Ը����Ӷ�����д�����ᱻ������

����д�������˶���ʱ�������̶߳Ը�����д���ᱻ�������Ӷ�����ɹ���

```C++
//����һ����д��
pthread_rwlock_t rwlock;
//�ڶ�֮ǰ�Ӷ���
pthread_rwlock_rdlock(&rwlock);

...do something about read

//����֮���ͷ���
pthread_rwlock_unlock(&rw_lock);

//д֮ǰ����д��
pthread_rwlock_wrlock(&rw_lock);

...do something write

//д֮���ͷ���
pthread_rwlock_unlock(&rw_lock);

//���ٶ�д��
pthread_rwlock_destroy(&rw_lock);

```


ͬ����д�����ڷ�����ģʽ��```trylock```
```C++
pthread_rwlock_tryrdlock(&rw_lock);
pthread_rwlock_trywrlock(&rw_lock);
```

##### ������
```C++
//��������������
pthread_spinlock_t spinlock;

//��ʼ��
pthread_spin_init(&spinlock, 0);

//����
pthread_spin_lock(&spinlock);

//����
pthread_spin_unlock(&spinlock);

//����
pthread_spin_destroy(&spinlock);
```
������pthread_spin_initʱ���ڶ�����Ϊpshared��int������ʾ�Ƿ�������̼乲��������Thread Process-Shared Synchronization����Ŀǰ�����������������������̼߳乲��
������Ҳ��������Ϊ���̼乲��pshared������ö��ֵ��

PTHREAD_PROCESS_PRIVATE����ͬ�����¶��߳̿���ʹ�ø�������
PTHREAD_PROCESS_SHARED����ͬ�����µ��߳̿���ʹ�ø�������

Linux�����߷ֱ�Ϊ0��1

##### ��������
���뻥��������ʹ��

pthread_cond_wait�л�Ե�ǰ�ӵĻ�����������Ȼ��ȴ�����֪ͨ��
pthread_cond_signal�л�֪֮ͨǰ�ȴ����̣߳�Ȼ��Ե�ǰ�ӵĻ����������𣿣�

**��ʹ�����������Ĺ����У�ʹ��while������if�����ж�״̬�Ƿ�����**
1. ��ֹ��Ⱥ������
2. ��ֹ��ٻ��ѣ�û��pthread_cond_signal�ͽ����������������

```C++
pthread_mutex_t mtx;
pthread_cond_t cond;

//��ʼ��
pthread_mutex_init(&mtx, NULL);
pthread_cond_init(&cond, NULL);

//����
pthread_mutex_lock(&mtx);


//�����ɹ����ȴ����������������˴�con��mtxһһ��Ӧ������bug
pthread_cond_wait(&cond, &mtx);

//����
pthread_mutex_lock(&mtx);

//��������������
pthread_cond_signal(&cond);


//����
pthread_mutex_unlock(&mtx);
//����
pthread_mutex_destroy(&mtx);
```

##### ����
pthread_cond_wait�л�Ե�ǰ�ӵĻ�����������Ȼ��ȴ�����֪ͨ��
pthread_cond_signal�л�֪֮ͨǰ�ȴ����̣߳�Ȼ��Ե�ǰ�ӵĻ����������𣿣�

**��ʹ�����������Ĺ����У�ʹ��while������if�����ж�״̬�Ƿ�����**
1. ��ֹ��Ⱥ������
2. ��ֹ��ٻ��ѣ�û��pthread_cond_signal�ͽ����������������


### ThreadBuffer
�̵߳Ĺ��û�����
#### ���ʣ�
Q:д���ʱ��Ϊʲô```bBuf```������ȫ����д�������ᵼ�¶���߳�д���ͻ�𣿣�
�����޸�```tagThreadBuff```��countʱȫ������д����������

A:������������ֻ�ǵ�����һ��������,û���Ƕ��̲߳���(����˵ʵ�ʳ����������),������ֻ��Ϊ��count����׼

### StreamProc
�����ݴ�����
��������ͷ����(SetLeadHead)ʱ,Ϊʲô��forѭ��������memcpy?

ͬʱÿ������ͷ,β����ʱ,���buffer��delete[]�ĳ�memset 0,�᲻�����ʡʱ��?

#### ����
�����ݰ��Ľṹ������ͷ��β��Сβ���庬��


### ThreadFrame
_Run������⣺
������ǰ����һ���׽����б�ʲô���ͣ�CLIENT��SERVER��ACCEPT��COMMON���������udp���ͣ�����״̬��FD_STATUS_COMMON����ʼ״̬����FD_STATUS_CONNECTED��FD_STATUS_CLOSED��FD_STATUS_LISTENED��FD_STATUS_TESTCONN�����У���Ҫ�������͡�����״̬���������

���У�CLIENT���͵��׽��֣����ܻ���COMMON��CONNECTED��CLOSED��TESTCONN��������Connect��״̬��ת�䣻

��SERVER���͵��׽��֣�ֻ��LISTENED״̬��

ACCEPT���͵��׽��֣�ָSERVER�����׽���accept�����������ӣ�



һ�����ȣ������׽��ּ���FD_SET���Ա�select

1���Ӷ���Ҫһ��fdlist��ȫ��������ڼ���FD_SETǰ:

    1���Դ�������CONNECTED״̬����fd���ԷǷ����׽��֣�״̬�޸�ΪCLOSE

    2�����ѹرգ�CLOSED����CLIENT�׽��֣����´�����״̬�޸�ΪCOMMON

    3���Դ��ڳ�ʼ����FD_STATUS_COMMON��״̬���׽��֣�

        a�������ѳ�ʱ��CLIENT�׽��֣���һ������Connect���������������޸��׽���״̬��
        	�����������Ͱѽ���״̬�����Զˣ�MOMϵͳ����
        	���ӷ���EWOULDBLOCK��˵������������δ��ɣ��ȴ��������TestConnect()
        	����ʧ�ܣ���close�׽��֣�
    
        b��δ��ʱ��CLIENT�׽��֣����������Ϊ�������ӻ���·��
    
        c��SERVER�����׽��֣���һ��listen��״̬�޸�ΪLISTENED

2������״̬���׽��־������������TESTCONN״̬���׽���ͬʱ����д����

3��selct�ȴ�������

������Σ����selectʱ�Ѿ����׽��֣���ͬ���Ͳ�ͬ������һ��fdlist������

1�����״̬ΪTESTCONN���׽��֣���д�����У���

����Ƿ����쳣״̬���ھ���(TestConnect())��������������ӣ�״̬�޸�ΪCONNECTED�����������״̬�޸�ΪCLOSE

2���������״̬�׽���

    1)SERVER�����׽��־���Accept�����µ��������׽��ּ���fdlist�У�����µ��������׽�����ACCEPT���͵�CONNECTED״̬��

    2)�ǳ�ʼ��COMMON״̬�ķ�SERVER�����׽��֣��������ݱ�recv��
    ���recv���ش���ģ��ر��׽��֣���״̬�޸�ΪCLOSE����������ΪACCEPT���׽���ֱ�����fdlist����CLIENT���͵��׽��ֲ���

����������select��ʱ��select return ==0��,������FD_STATUS_TESTCONN״̬���׽��֣�Ҳֻ�д�״̬���׽���select��ʱ��Ҫ���������û�г���MAX_CONNECT_TIMEʱ�䣬�о�CLOSE����û�оͲ���Ҫ�ܣ�������fdlist������



```TestConnect��&rset, &wset��```����:

�����select��getsockopt�ķֲ���һ�����select�У���ĳ��fd���е�����getsockopt���error��TestConnect���ֶ�ĳfd����getsockopt��



#### ����

������connect�����getsockopt���ص�error��=0��һ������Ϊ���ӳ�ʱ��

ѧϰ�ص�����CallBack()

������connect������ʱ����ö�д������⣡

### Tips
1. typedef����using ���ں���ָ�������ӳ���ɶ���

```C++
//fpΪ����ָ��
typedef int(*fp)(int,float);
//using fp = int(*)(int, float)

//�������Ϻ���ָ��
fp func(int a){}

//���û��typedef���ɶ��Խϲ��Ҫ������������ܿ�����ʲô
 int (*func(int a))(int, float)
{}
```


2. �̴߳���

```int pthread_create(pthread_t *threaed, const pthread_attr_t *attr, void *(*start_routine)(void *), void *arg);```

�����̲߳�ִ�к�����
��������start_routineΪ�̴߳�������Ҫִ�еĺ���������ֵ�Ͳ������;�Ϊvoid *�����Դ������ʱ������Ҫǿת��
���Ĳ���Ϊstart_routine�����Ĳ�����Ϣ��

3. �ȴ��߳�ִ�н���

```int pthread_join(pthreaed_t thread, void ** retval);```

����**���������߳�**��ֱ��**Ŀ���߳�**ִ����ϣ����յ�����ֵ����ִ����Ϻ�thread�߳���ռ��ռ��Դ�ᱻ�ͷ�
thread��Ϊ��Ҫ�ȴ����̣߳�
retvalΪ�̵߳ķ���ֵ�����threadû�з���ֵ���߲���Ҫ���ܷ���ֵ����������ΪNULL