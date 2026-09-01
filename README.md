## 문제 1. 배달 로봇의 연산 분담과 실시간성 설계

1. **연산 분담 배치표**
   
| 작업               | 위치            | 지연 예상        | 데이터량         | 근거                                                                                              |
| ---------------- | ------------- | ------------ | ------------ | ----------------------------------------------------------------------------------------------- |
| 모터 속도            | 임베디드          | 1ms ~ 5ms    | 몇 Byte(주기적)  | 대역폭이 작음. 지연 시간이 길어지면 진동, 엔코더 피드백 실시간성이 필요하다.                                                    |
| 장애물 감지           | 임베디드          | 50ms ~ 100ms | 1 ~ 2MB/s    | 라이다 데이터 및 IMU를 기반으로 비상 정지나 회피를 수행. 오프라인 상황에서 안전을 보장하기 위해 지연예산 내 온보드 처리 필수.                      |
| 보행자 인식           | Edge AI       | 30ms ~ 60ms  | ~ 100MB/s 이상 | RGB카메라 데이터 100MB 이상을 클라우드로 전송하면 과도한 비용 발생. NPU, GPU 가속을 이용해 내부에서 실시간 인지 수행.                     |
| 지도 기반 <br>경로 계획  | Edge AI, 클라우드 | 200ms ~ 1s   | KB/s 수준      | 로봇이 움직이는 동안 갱신 되어야 한다. Edge AI에서 지도와 현재 위치·장애물 정보를 이용해 계산하면 온보드 매핑 데이터를  활용할 수 있다.              |
| 배달 완료 <br>사진 업로드 | 클라우드          | 몇 초, 분       | 사진1장 1~3MB   | LTE를 사용. 이벤트 발생 시점에만 고화질 사진을 클라우드로 전송하여 고객 확인 및 기록 저장용으로 활용. 오프라인시 로컬에 저장했다가 재전송.               |
| 운행 로그 집계         | 클라우드          | 몇 초, 분       | KB ~ MB(가변)  | 상태 로그(센서, 모터 상태 등)는 제어 루프에 영향을 주지 않으므로 로컬 버퍼에 저장, 압축/요약하여 주기적으로 클라우드로 전송. 클라우드에서 데이터를 집계·분석합니다. |



1. **카메라 원시 영상 전송량**: 
   해상도 1920 x 1080 = 2,073,600 pixels
   2,073,600(pixels) x 3bytes(RGB) =  6,220,800 bytes
   초당 데이터 량 = 6.2208MB x 30 frames = 186.62MB
   네트워크Mbps : 186.62 x 8 bit = 1,493Mbps(1.49 Gbps)
   Raw데이터 전송 요구량(1,500Mbps)이 LTE평균 대역폭(30Mbps)를 50배 이상 초과함
   
   영상압축 등을 거치더라도 인코딩, 디코딩, 전송지연, 클라우드 지연, 오프라인 등 때문에 인식지연 인지 시스템이 안전을 보장 할 수 없다.
   


2. **인지·판단·제어 계층 매핑과 주기표**

| 계층  | 작업                             | 위치      | 갱신 주기 | 데이터 흐름 및 특징                                      |
| --- | ------------------------------ | ------- | ----- | ------------------------------------------------ |
| 제어  | 모터 조향·가감속 제어, 액추에이터 명령         | 임베디드    | 1ms   | 저지연·주기적 처리 필요. 목표 속도 명령과 엔코더 피드백을 받아 모터 전압 제어    |
| 인지  | 카메라 영상 수집, 객체/장애물 검출, 위치·상태 추정 | Edge AI | 33ms  | RGB 카메라 영상 기반 보행자 및 위치 추정, 대용량 센서 데이터 → 압축/특징 추출 |
| 판단  | 상황 판단, 경로 계획, 행동 선택, 위험도 평가    | 클라우드    | 100ms | 상대적으로 작은 상태 정보<br>인지 단의 장애물/보행자 정보를 받아 경로 재설계    |
   센서 데이터 
    │ 
    │ 30 Hz
     ▼
 ┌─────────┐
 │              인지                │ 
 │        Perception        │ 
 │ 객체·차선·위치 추정  │ 
 └─┬───────┘ 
    │ 
    │ 10 Hz 
    │ 객체/상태 정보 
    ▼ 
┌─────────┐ 
│               판단                │ 
│ Decision / Planning │ 
│ 경로·행동·목표속도    │ 
└─┬───────┘ 
    │ 
    │ 50~100 Hz 
    │ 목표 경로/제어 
    ▼ 
┌─────────┐ 
│               제어                │ 
│            Control            │ 
│     조향·가감속·제동    │ 
└─┬───────┘ 
    │ 
    ▼ Actuator 
    │ 
    └─ 상태 ──► 제어/인지



4. **Hard / Firm / Soft 분류표** 

| 모터 속도제어 | 장애물 감지 | 보행자 인식 | 지도 경로 계획   | 배달 완료<br>사진 업로드 | 운행 로그 집계 |
| ------- | ------ | ------ | ---------- | --------------- | -------- |
| Hard    | Hard   | Firm   | Firm, Soft | Soft            | Soft     |
   Hard 실시간
   Hard 작업의 마감 실패는 단순히 데이터가 늦게 도착하는 문제가 아니라 물리적 결과로 이어진다. 모터 속도 제어가 늦으면 목표 속도 추종이 깨지고, 장애물 감지가 늦으면 제동·회피 시작 자체가 늦어져 **충돌 가능성**이 생깁니다.
   모터 토크가 급격하게 요동치며 진동(Oscillation)이 발생하고, 시설물과 물리적으로 직접 충돌할 수 있으며 로봇이 균형을 잃고 전복될 수 있습니다.
   
   
   
5. **주기 · 지연 · 지터 구분** 
   주기 : 모터 속도 제어를 일정한 시간 간격(10ms)마다 실행하는 것처럼, 작업이 반복 실행되는 시간 간격이다.
   지연 : 장애물 감지 결과를 받아 모터에 정지 명령을 내리기까지 데이터 처리, 응답에 걸리는 소요시간(50ms)을 말한다.
   지터 : 모터나 센서가 주기적으로 들어와야 하는 데이터가 스케쥴링이나 다른 요인으로 실행 간격이 달라지는 현상(주기의 변동성)을 말한다.




## 문제 2. 원격 접속(SSH)과 센서 장치 경로 고정

1. **고른 접속 대상**: `localhost`
   내 서버 ssh를 설치 한다.(openssh)
   ```sudo apt install openssh-server```
   **active(running) 확인**
   ```systemctl status ssh```
   ```
   ssh.service - OpenBSD Secure Shell server
   Loaded: loaded (/lib/systemd/system/ssh.service; enabled; vendor preset: enabled)
   Active: active (running)
   ```
   **22번 포트 확인**
   ```ss -tlnp | grep :22```
   **localhost로 접속**
   ```ssh 사용자@localhost```
   ```
   Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 6.8.0-138-generic x86_64)
   ```
   ```
   pa8@pa8-Legion-Pro-5-16IAX10:~$ who
   pa8      tty2         2026-09-01 09:04 (tty2)
   pa8      pts/3        2026-09-01 13:42 (127.0.0.1)
   pa8@pa8-Legion-Pro-5-16IAX10:~$ echo $SSH_CONNECTION
   127.0.0.1 51124 127.0.0.1 22
   ```
   
   
2. **개인키·공개키 중 서버에 등록하는 것**: 
   서버에 올라가는 것은 공개키이고, 개인키는 기기에만 보관하므로 공개키가 노출되어도 개인키 없이는 서명 위조나 복호화가 불가능해 안전.
   
   
3. **원격 단일 명령 실행과 `scp` 전송 출력**
   접속하지 않고 명령만 실행
   ```
   ssh 사용자@서버 'uname -a
   ```
   ```
   ssh pa8@localhost 'uname -a'
   Linux pa8-Legion-Pro-5-16IAX10 6.8.0-138-generic #138~22.04.1-Ubuntu SMP
   PREEMPT_DYNAMIC Fri Aug  7 13:43:15 UTC  x86_64 x86_64 x86_64 GNU/Linux
   ```
   파일 복사
   ```
   scp 파일 사용자@아이피:절대위치
   ```
   ```
   pa8@pa8-Legion-Pro-5-16IAX10:~/fake_sensors$ scp hi.txt pa8@localhost:/home/pa8/
   hi.txt                                                                                   100%    0     0.0KB/s   00:00
   ```
   
   시리얼 장치 확인
   ```
   ls -l /dev/tty*
   ```
   ```
   pa8@pa8-Legion-Pro-5-16IAX10:~$ ls -l /dev/tty*
   crw--w---- 1 root tty     4, 19 Sep  1 09:04 /dev/tty19
   crw--w---- 1 pa8  tty     4,  2 Sep  1 09:04 /dev/tty2
   crw------- 1 root root    5,  3 Sep  1 09:04 /dev/ttyprintk
   crw-rw---- 1 root dialout 4, 64 Sep  1 09:04 /dev/ttyS0
   ```
   
   
4. **두 장치를 구분한 속성**: 라이다 `___` / IMU `___`
   가상 sensor 장치 생성
   ```
   mkdir -p ~/fake_sensors && cd ~/fake_sensors
   ```
   파일생성 10M 크기로
   ```
   truncate -s 10M lidar.img imu.img
   ```
   블록장치로 연결
   ```
   sudo losetup -f --show lidar.img
   sudo losetup -f --show imu.img
   ```
   udev데몬의 단계별로 속성 출력
   ```
   udevadm info --attribute-walk /dev/loop(N)
   ```
   ```
   pa8@pa8-Legion-Pro-5-16IAX10:~/fake_sensors$ udevadm info --attribute-walk /dev/loop22
   looking at device '/devices/virtual/block/loop22':
    KERNEL=="loop22"
    SUBSYSTEM=="block"
    DRIVER==""
   ```
   
   
5. **작성한 udev 규칙 2개** + **규칙 키 설명표**
   
   규칙파일 수정
   /etc/udev/rules.d/99-robot-sensor.rules
   ```
   # virtual robot sensor rules
   
   # LiDAR
   SUBSYSTEM=="block", KERNEL=="loop22", SYMLINK+="robot_lidar", MODE="0666"
   
   # IMU
   SUBSYSTEM=="block", KERNEL=="loop23", SYMLINK+="robot_imu", MODE="0666"
   ```

| 규칙        | 설명                                                                                                              |
| --------- | --------------------------------------------------------------------------------------------------------------- |
| SUBSYSTEM | 장치가 속한 udev 서브시스템. 블록 장치=block, USB=tty                                                                         |
| KERNEL    | 커널이 부여한 장치 이름 패턴을 조건으로 비교합니다.                                                                                   |
| ATTR{...} | sysfs의 장치 속성을 읽어 조건으로 비교하거나 값을 설정합니다. 실제 USB 센서는 ATTRS{idVendor}, ATTRS{idProduct}, ATTRS{serial}처럼 식별에 주로 씁니다. |
| SYMLINK   | 기존 장치는 유지하면서, 같은 장치를 가리키는 추가 심볼릭 링크.                                                                            |
| MODE      | 장치 파일의 권한을 설정합니다. 일반적으로 그룹에 읽기·쓰기 권한을 주기 위해 "0660"을 사용합니다.                                                      |
| GROUP     | 장치 파일의 소유 그룹을 설정합니다. 해당 그룹에 사용자를 넣으면 sudo 없이 장치에 접근하게 할 수 있습니다.                                                 |
| ==        | **비교 연산자**. 장치의 현재 속성이 오른쪽 값과 일치할 때만 규칙이 적용됩니다.                                                                 |
| =         | **값 설정 연산자**. 해당 키의 값을 지정합니다. 목록 성격의 키에서는 기존 값을 대체할 수 있습니다.                                                     |
| +=        | **추가 연산자**. 기존 값은 유지하고 새 항목을 덧붙입니다. SYMLINK에는 보통 이것을 사용합니다.                                                     |

   
6. **순서를 바꿔 재연결한 뒤 `ls -l /dev/robot_*` 결과**
   ```
   ls -l /dev/robot_*
   lrwxrwxrwx 1 root root 6 Sep  1 14:17 /dev/robot_imu -> loop23
   lrwxrwxrwx 1 root root 6 Sep  1 14:16 /dev/robot_lidar -> loop22
   ```
   
   ```
   sudo losetup -d lidar.img
   sudo losetup -d imu.img
   sudo udevadm trigger
   sudo losetup -f --show imu.img
   sudo losetup -f --show lidar.img
   sudo udevadm trigger
   ls -l /dev/robot_*
   lrwxrwxrwx 1 root root 6 Sep  1 15:10 /dev/robot_imu -> loop23
   lrwxrwxrwx 1 root root 6 Sep  1 15:10 /dev/robot_lidar -> loop22
   ```
   
   
   
7. **실제 USB 센서용 규칙 초안과 구분 근거**
   
   실제 USB 센서용 규칙 초안
  ```
  # 라이다 설정
  SUBSYSTEM=="tty", ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6001", MODE="0666", SYMLINK+="robot_lidar"
  # IMU 설정
  SUBSYSTEM=="tty", ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6015", MODE="0666", SYMLINK+="robot_imu"
  ```
   
   동일안 idVendor를 갖고 idProduct만 다른 상황에서 리눅스 커널에서 vendor와 product모두 비교하여 분류. idProduct까지 같다면 Serial번호, 포트 번호까지 구분하여 분류를 할 수 있다.
   


A브랜치 수정 내용과 B브랜치 수정 내용을 병합합니다.
