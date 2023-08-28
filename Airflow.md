# GCP 환경에 Airflow를 설치하고 분산크롤링하기

초기엔 그저 여러사이트에서 기사를 수집하는 작업을 airflow를 통하여 진행하려고했으나, 여러사이트의 기사를 수집해오다보니 생각보다 시간이 꽤 오래걸리는점이 자꾸 마음에걸렸다.

그래서 GCP환경에 Airflow를 설치하고 여러대의 인스턴스에 크롤링작업을 분산시켜서 시간을 단축시키면 어떨까?라는 생각을 하게되었다.

그리하여 GCP환경에 Airflow를 통한 분산 크롤링작업을하는 과정을 기록으로 남기려한다.

총 3개의 인스턴스를 사용할것이고, 하둡과 마찬가지로 마스터-워커1-워커2의 구조를 가질것이다. 그리고 전처리를 통해 mysql과 하둡에 적재하는것까지 해볼것이다.


## GCP에 Airflow 설치

일단 airflow를 설치해야하니 airflow를 인스턴스에 설치할것인데 타 라이브러이와의 충돌방지를위해 가상환경에 설치하기를 권고하는듯하다. <u>그러므로 conda를 이용하여 가상환경을 만들어줄것이다.</u>

[아나콘다 공식 홈페이지](https://www.anaconda.com/download#downloads)


```bash

wget https://repo.anaconda.com/archive/Anaconda3-2023.07-2-Linux-x86_64.sh

GCP 인스턴스가 Ubuntu20 -x86을 사용중이니 위의 버전을 다운로드받았다.
.sh파일을 다운로드받았으면 권한을 변경해주고 실행한다

chmod 755 Anaconda3-2023.07-2-Linux-x86_64.sh

sh Anaconda3-2023.07-2-Linux-x86_64.sh or ./ Anaconda3-2023.07-2-Linux-x86_64.sh

실행하면 라이센스에대한 안내가뜨고, 엔터키를 눌러 마지막으로가면 동의 안내가 뜬다.

    Do you accept the license terms? [yes|no]
    >yes

이후 설치 경로에대해 안내가나온다. 엔터를 누른다.

    [/home/usr/anaconda3] >>>

그러면 설치경로대로 경로가 알아서 설정된다.


설치가 끝난후 bashrc에 들어가보면 아나콘다가 설치되면서 자동으로 등록된 환경변수가있다. 적용시켜주자.

source ~/.bashrc


적용이 끝났으면 가상환경을 하나 만들어주고 파이썬 버전을 지정해주자.

conda create -n [가상환경이름] python=[원하는버전]

나는 3.9버전을 사용할것이다.

conda create -n conda_airflow python=3.9 -y

가상환경을 활성화시켜주자

conda activate conda_airflow

# airflow
export PATH=$PATH:/home/계정명/.local/bin

▲ 설치하기전에 경로를 지정해줬다.

이제 진짜 에어플로우를 설치해주자. 
단 에어플로우는 파이썬과 버전이안맞으면 오류를 일으키니, 버전에 맞는 에어플로우를 설치해줘야한다.

pip install apache-airflow

-> 버전이 안맞으면 새빨간줄로 오류가뜨니까 잘읽어보고 버전에 맞는걸 설치해주자.

```
## (선택)GCP 인스턴스의 용량을 늘리고 디스크를 추가해서 사용해보자

설치하다가보니 GCP의 용량이 다 찬걸 확인했다.
생각보다 아나콘다가 정말 아나콘다급으로 매우 많은 용량을 차지하고있었다..

df -h 를 통해 디스크를확인해보니 used가 100%였다.
현재 작업을 진행하는 마스터인스턴스를 카피해서 새로만들자니 내부IP가 고정이아니여서, 새로운 인스턴스로 갈아타면 다시 설정해줘야하는 작업을해야하기때문에 귀찮았다..

그래서 영구디스크의 크기를 늘리고, 여유롭게 보조디스크를 하나 연결하는게 좋겠다는 생각을해서 사용중인 GCP인스턴스의 영구디스크 크기를 늘리는법과 새 디스크를 추가하였다.

[GCP영구디스크 크기 늘리기](https://cloud.google.com/compute/docs/disks/resize-persistent-disk?authuser=7&hl=ko&_ga=2.237255808.-1899439076.1677074651)

<mark>다만, 인스턴스의 영구디스크는 늘리는건되지만 줄이는건 안된다고한다.</mark>

이것을 사용하기위해 포맷을해줘야하니 포맷을 진행해주는데 포맷진행법은 리눅스와같다.

    $ sudo lsblk
    
    NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
    loop0     7:0    0  55.7M  1 loop /snap/core18/2785
    loop n ....
    lopp n...

    sda       8:0    0    20G  0 disk
    ├─sda1    8:1    0  19.9G  0 part /
    ├─sda14   8:14   0     4M  0 part
    └─sda15   8:15   0   106M  0 part /boot/efi
    sdb       8:16   0    10G  0 disk
    
    위의 명령어를 입력하면 디스크들이나오는데 저기 맨아래의 sdb가 새로추가한 디스크다.
    아래의 명령어를 차례로 입력해주자
    
    #HDD를 EXT4로 포맷
    sudo mkfs.ext4 -m 0 -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/sdb

    # 마운트 포인트 생성

    sudo mkdir -p /mnt/disks/data

    #HDD 마운트
    sudo mount -o discard,defaults /dev/sdb /mnt/disks/data


    #HDD 권한추가

    sudo chmod a+w /mnt/disks/data

    # 마운트 확인

    df -H | grep /mnt/disks/data
    /dev/sdb         11G   25k   11G   1% /mnt/disks/data



### VM이 다시 시작할때 자동마운트하게만들자

    # fstab 백업파일 만들기
    
    sudo cp /etc/fstab /etc/fstab.backup

    # /etc/fstab에 기입할 UUID확인

    sudo blkid /dev/sdb

    그러면 UUID가 UUID="a0alskd1-31sdk-..." TYPE="ext4" 와 같은 포맷으로나오는데
    ""안의 UUID의 VALUE값만 복사.

    # /etc/fstab를 열어 맨 아래에 아래 uuid와 아래내용 붙여넣기

    sudo vi /etc/fstab

    UUID=UUID_VALUE(아까복사한값) /mnt/disks/data ext4 discard,defaults,nofail 0 2​

    # 재부팅후에도 제대로 마운트를하고있는지 /dev/sdb 마운트 확인

    sudo df -h

    Filesystem      Size  Used Avail Use% Mounted on
    /dev/root        20G   11G  8.5G  57% /
    devtmpfs        3.9G     0  3.9G   0% /dev
    tmpfs           3.9G     0  3.9G   0% /dev/shm
    tmpfs           794M 1004K  793M   1% /run
    tmpfs           5.0M     0  5.0M   0% /run/lock
    tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
    /dev/loop0       56M   56M     0 100% /snap/core18/2785
    loop n ...         ..  ..     ..    . ...
    ...
    /dev/sdb        9.8G   24K  9.8G   1% /mnt/disks/data

    위와같이 재부팅해도 디스크가 잘연결되어있는걸 볼수있다.


참고

[gcp디스크 추가](https://yooloo.tistory.com/m/156)

```bash

이제 다시 에어플로우를 설치해주자

pip install apache-airflow

나는 MysqlDB와 에어플로우를 연동시킬거니까 관련 패키지도 미리 설치해줄것이다.

sudo apt install libmysqlclient-dev -y

pip install apache-airflow-providers-mysql

```