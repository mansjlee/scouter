# president 어플리케�?�션 JBoss EAP 7.4 컨테�?�너화


## 1. jboss eap 7.4 �?�미지 구성


### 1.1. �?�미지를 disconnected �?서 준비하기
```
# �?�터넷환경�?서 다운받�?� jboss �?�미지를 disconnected 환경 서버�?(bastion) 로딩
$ podman load jboss7.4-jdk8.tar
$ podman images
..
registry.redhat.io/jboss-eap-7/eap74-openjdk8-openshift-rhel7  7.4.11-3  ...
..

# 서버�? 로딩�?� �?�미지를 private registry �? push 하기
$ podman push registry.redhat.io/jboss-eap-7/eap74-openjdk8-openshift-rhel7:7.4.11-3
```


### 1.2. 사용�?환경�? 맞는 Dockerfile �?성

준비�?� �?�미지를 사용�?�?� 환경�? 맞춰 변경하�?��? 선언합니다.

```
# 임�?��?� build 디렉토리
$ cd /root/apps/president

# Dockerfile �? COPY �? RUN 등�?� 사용하여 사용�?�?� 환경으로 변경
$ vi Dockerfile
FROM registry.redhat.io/jboss-eap-7/eap74-openjdk8-openshift-rhel7:7.4.11-3

COPY bin/* /opt/eap/bin/
COPY configuration/* /opt/eap/standalone/configuration/
COPY sso /opt/eap/sso

USER 0
RUN chown -R jboss /opt/eap/standalone/configuration/ /opt/eap/bin/ /opt/eap/sso/ && \
    chmod -R 774 /opt/eap/standalone/configuration/ /opt/eap/bin/ /opt/eap/sso/safeagent/bin/

ENV LANG=ko_KR.UTF8 TZ=Asia/Seoul

USER 185
```


### 1.3. 사용�?환경�? 맞춰 변경할 파�?� 준비

```
# /opt/eap/bin �? 추가�?� 파�?�들 확�?�
$ cd bin
$ ls -1
env.sh
env.properties
standalone.sh
openshift-launch.sh

# /opt/eap/sso �? 추가�?� 디렉토리들 확�?�
$ cd /root/apps/president/sso
$ ls -1 */*
safeagent/bin
safeagent/conf
..

# sso agent xstart 수정내용 확�?�
$ cd safeagent/bin
$ diff xstart xstart.org
...

# sso agent secure.conf 수정내용 확�?�
$ cd ../conf
$ diff safeagent.conf safeagent.conf.org
...
```


### 1.4. 배�?�할 어플리케�?�션 war 확�?�

```
# 디플로�?� war 디렉토리
$ cd /root/apps/president/
$ ls .git
$ cd deployments/ROOT.war

# DB IP 등 설정 확�?�
$ cat WEB-INF/classes/FTeGV/Props/globals.properties
...

# /app/kotrawas4/domains2/PD2/deployments/president-1.0.0.war/WEB-INF/upload/ 절대경로 참고 소스 확�?�
$ find . -type f -exec grep WEB-INF/upload {} \;
...
```


### 1.5. git 명령으로 source 관리

gitlab �?� disconnected 환경�? 구성�?�어 있다고 가정합니다. gitlab 구성�?� 레드햇�?� 지�?범위가 아닙니다.

```
# git 소스를 20230626 �?��?�는 commit 메세지와 함께 push 하는 예시
$ cd /root/apps/president/
$ git add .
$ git commit -m 20230626
$ git push
```


## 2. jboss eap 7.4 구성


### 2.1. jboss eap 7.4 기본 template 변경 �? 배�?�

템플릿�?는 빌드와 배�?� 관련�?� 모든 �?체들�?� 미리 선언�?�어 있습니다. 재사용�?� 편리하게 해�?니다.

```
# https://github.com/jboss-container-images/jboss-eap-openshift-templates/tree/eap74/templates
# 다운받�?� 기본 템플릿�?� Maven �? Galleon �?� 필요한 환경�? 맞춰져 있으므로, 변경�?� 필요합니다.
$ wget https://raw.githubusercontent.com/jboss-container-images/jboss-eap-openshift-templates/eap74/templates/eap74-basic-s2i.json
$ diff eap74-basic-s2i.json eap74-custom-s2i.json
...

# 수정한 eap74-custom-s2i.json 를 �?�러스터�? 배�?�
$ oc create -f eap74-custom-s2i.json

$ oc get template
NAME               DESCRIPTION                                                                        PARAMETERS     OBJECTS
eap74-custom-s2i   An example JBoss Enterprise Application Platform application. For more inform...   16 (4 blank)  
```


### 2.2. 네임스페�?�스 환경 구성

```
# 네임스페�?�스 �?성
$ oc new-project president-poc

# 네임스페�?�스로 �?��?�
$ oc project president-poc
```


### 2.3. 빌드 기본 �?�미지스트림 �?성

```
# eap74-openjdk8-openshift-rhel7:7.4.11-3 �?�미지스트림 �?체 �?성
$ oc import-image eap74-openjdk8-openshift-rhel7:7.4.11-3 --from registry01.pen.ito.com:5000/jboss-eap-7/eap74-openjdk8-openshift-rhel7:7.4.11-3 --confirm

$ oc get is
NAME                             IMAGE REPOSITORY                                                         TAGS        UPDATED
eap74-openjdk8-openshift-rhel7   registry01.pen.ito.com:5000/jboss-eap-7/eap74-openjdk8-openshift-rhel7   7.4.11-3    12 days ago

```


### 2.4. new-app 으로 빌드 �? 배�?� 환경 만들기

buildconfig, imagestream, service, route, deploymentconfig 등�?� 주로 �?체들�?� 템플릿�?� 통해 �?성합니다.

```
# 템플릿과 git source 등�?� �?�용하여 새로운 S2I CI/CD 를 �?성하는 명령
$ oc new-app --name=poc --template=eap74-custom-s2i -p IMAGE_STREAM_NAMESPACE=president-poc -p EAP_IMAGE_NAME=eap74-openjdk8-openshift-rhel7:7.4.11-3 -p SOURCE_REPOSITORY_URL=http://registry01.pen.ito.com:9080/root/president

# 템플릿으로 배�?��?� �?체들 확�?�
$ oc get all
...

# 컨테�?�너�?� 주요 Resource 설정 �? Env 등�?� deploymentconfig �?서 확�?�할 수 있습니다.
$ oc get dc/poc -oyaml
...
```


### 2.5. git source 변경 트리거 설정(optional)

```
# 웹후�?�를 gitlab �? 설정
# buildconfig �?� spec.triggers.generic.secret �?� secret 값 확�?�
$ oc get bc poc -oyaml

# president-poc �?� poc buildconfig �? 대한 주소 예시: https://api.apps.pen.ito.com:6443/apis/build.openshift.io/v1/namespaces/president-poc/buildconfigs/poc/webhooks/<secret>/generic
$ echo https://api.apps.pen.ito.com:6443/apis/build.openshift.io/v1/namespaces/president-poc/buildconfigs/poc/webhooks/$(oc get bc poc -ojsonpath='{.spec.triggers[1].generic.secret}')/generic

# gitlab registry01.pen.ito.com:9080/root/president 브�?�우저�?서 Settings -> webhook 로 �?��?�. 위�?� 주소를 넣고 add webhook 수행
# �?�후부터 git source 변경시, deploymentconfig �? �?�?� Trigger 전달�?�

```


## 3. war 빌드 지연 완화


### 3.1. war �?서 대량�?� contents 를 �?�함하는 디렉토리들�?� 분리

컨테�?�너 �?�미지를 빌드하는 과정�?서 source 파�?�들�?� 합치는 부하를 줄�?�기 위한 작업입니다.

```
$ cd /root/apps/president/deployments/ROOT.war

# 25G 를 �?유한 upload 디렉토리를 1차 분리
$ mv ./WEB-INF/upload /mnt/president-pv

# 1G �?��?�?� �?유하는 bzguide,static,upload 디렉토리를 추가 2차 분리
$ mv ./bzguide ./static ./upload /mnt/president-images/
```


### 3.2. 분리�?� 디렉토리를 persistent volume 으로 �?성

주�?�: bastion �? nfs 서버로 임시 구성하여 테스트하였습니다. RHEL nfs 서버는 OpenShift �?� 권장 구성�?� 아닙니다. nfs 서버구성�?� 레드햇�?� 지�?범위가 아닙니다.
참고: https://docs.openshift.com/container-platform/4.12/scalability_and_performance/optimization/optimizing-storage.html#specific-application-storage-recommendations

```
# upload 용 nfs PV �? PVC �?성 (1차)
$ oc create -f - <<EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: president-pvc
  namespace: president-poc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: registry-pv
spec:
  capacity:
    storage: 50Gi
  accessModes:
  - ReadWriteMany
  nfs:
    path: /mnt/president-pv
    server: 192.168.50.176
  persistentVolumeReclaimPolicy: Retain
  claimRef:
    name: president-pvc
    namespace: president-poc
EOF

# bzguide,static,upload 용 nfs PV �? PVC �?성 (2차)
$ oc create -f - <<EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: president-image-pvc
  namespace: president-poc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: registry-images
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteMany
  nfs:
    path: /mnt/president-images
    server: 192.168.50.176
  persistentVolumeReclaimPolicy: Retain
  claimRef:
    name: president-image-pvc
    namespace: president-poc
EOF
```


### 3.3. deploymentconfig �? 추가할 PVC 들�?� mount

참고: eap74-custom-s2i.json 템플릿�?는 해당 내용�?� �?�미 반�?�?�어 있습니다.

```
# deploymentconfig �?체�? 위�?서 �?성한 PVC 들 mount 구문 추가
# 절대경로 주�?� - /app/kotrawas4/domains2/PD2/deployments/president-1.0.0.war/WEB-INF/upload/
$ oc edit dc/pod
..
            volumeMounts:
            - name: upload
              mountPath: /app/kotrawas4/domains2/PD2/deployments/president-1.0.0.war/WEB-INF/upload/
            - name: president-image-pvc
              mountPath: /opt/eap/standalone/deployments/ROOT.war/bzguide
              subPath: bzguide
            - name: president-image-pvc
              mountPath: /opt/eap/standalone/deployments/ROOT.war/static
              subPath: static
            - name: president-image-pvc
              mountPath: /opt/eap/standalone/deployments/ROOT.war/upload
              subPath: upload
..
          volumes:
          - name: upload
            persistentVolumeClaim:
              claimName: president-pvc
          - name: president-image-pvc
            persistentVolumeClaim:
              claimName: president-image-pvc
..

```










