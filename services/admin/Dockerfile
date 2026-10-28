# admin-service 컨테이너 이미지
#
# 왜 필요한가:
#   지금 admin-service 는 담당자 PC 의 IntelliJ 에서만 돈다. 그런데 모든 화면이 로그인을
#   거치고 로그인은 admin-service 를 타기 때문에, 이 PC 가 꺼지면 팀 전체가 아무 화면도
#   못 쓴다. 이미지로 만들어 원격 서버에 올려야 그 의존이 사라진다.
#
# 만드는 법 / 실행하는 법은 맨 아래 주석 참고.


# ══════════════ 1단계: 빌드 ══════════════
# jar 를 만드는 단계다. 여기서 쓰는 Gradle·JDK 는 최종 이미지에 남지 않는다.
FROM eclipse-temurin:17-jdk AS build
WORKDIR /app

# 의존성 정의 파일만 먼저 복사한다.
# 소스만 고쳤을 때 라이브러리를 다시 내려받지 않기 위해서다.
# Docker 는 파일이 안 바뀐 단계는 건너뛰는데, src 를 먼저 복사해버리면
# 코드 한 줄만 고쳐도 의존성부터 전부 다시 받는다.
COPY gradlew gradlew.bat ./
COPY gradle ./gradle
COPY build.gradle settings.gradle ./
RUN chmod +x gradlew && ./gradlew dependencies --no-daemon

# 이제 소스를 복사하고 jar 를 만든다.
# -x test : 테스트를 건너뛴다. 테스트가 Oracle·Redis 에 붙으려 하면
#           이미지 빌드 중에는 그 주소들이 없어서 실패한다.
COPY src ./src
RUN ./gradlew bootJar --no-daemon -x test


# ══════════════ 2단계: 실행 ══════════════
# 실제로 서버에서 도는 이미지다. JDK 가 아니라 JRE 를 쓴다.
# 컴파일 도구가 필요 없어서 용량이 절반 이하로 줄고, 공격 표면도 작아진다.
FROM eclipse-temurin:17-jre
WORKDIR /app

# root 로 돌리지 않는다. 컨테이너가 뚫렸을 때 할 수 있는 일을 줄인다.
RUN useradd --create-home --shell /bin/false appuser
USER appuser

# 1단계에서 만든 jar 만 가져온다. 소스도 Gradle 도 안 넘어온다.
COPY --from=build /app/build/libs/adminservice-0.0.6.jar app.jar

EXPOSE 9191

ENTRYPOINT ["java", "-jar", "app.jar"]


# ══════════════ 사용법 ══════════════
#
# 이미지 만들기 (이 파일이 있는 폴더에서)
#   docker build -t adminservice:0.0.5 .
#
# 실행하기
#   설정은 이미지에 굽지 않고 실행할 때 넣는다. 개발표준가이드 5-2 절 내용이다.
#   그래야 주소가 바뀌어도 코드를 고치고 다시 커밋할 필요가 없다.
#
#   docker run -d --name adminservice --restart unless-stopped -p 9191:9191 \
#     -e SPRING_DATA_REDIS_HOST=192.168.1.126 \
#     -e REDIS_PASSWORD=비밀번호 \
#     -e SEAWEED_ENDPOINT=http://192.168.1.126:8333 \
#     adminservice:0.0.5
#
#   환경변수 이름 규칙: application.properties 의 키를 대문자로 바꾸고
#   점(.)과 하이픈(-)을 밑줄(_)로 바꾸면 된다. Spring Boot 가 알아서 매핑한다.
#     spring.data.redis.host  ->  SPRING_DATA_REDIS_HOST
#     seaweed.endpoint        ->  SEAWEED_ENDPOINT
#
# 로그 보기
#   docker logs -f adminservice
#
# 재배포 (코드 고친 뒤)
#   docker build -t adminservice:0.0.1 .
#   docker rm -f adminservice
#   docker run ... (위와 동일)
#
#   ※ 공용 서버에서는 docker-compose down 처럼 전체를 내리는 명령을 쓰지 말 것.
#     다른 팀 서비스까지 같이 멈춘다. 항상 서비스 이름을 지정한다.
