ARG BASE_IMAGE="eclipse-temurin:11-jdk"
ARG EXT_BUILD_COMMANDS=""
ARG EXT_BUILD_OPTIONS=""

# Etapa de build do overlay
FROM eclipse-temurin:11-jdk AS overlay

# Instala Git e dependências
RUN apt-get update && \
    apt-get install -y git ca-certificates && \
    apt-get clean

# Clona o projeto base do CAS
RUN mkdir -p /tmp/cas-overlay && \
    git clone -b 6.6 --single-branch https://github.com/apereo/cas-overlay-template.git /tmp/cas-overlay

# Copia arquivos do overlay
RUN mkdir -p /cas-overlay && \
    cp -r /tmp/cas-overlay/src /cas-overlay/src && \
    cp -r /tmp/cas-overlay/gradle /cas-overlay/gradle && \
    cp /tmp/cas-overlay/gradlew /cas-overlay/ && \
    cp /tmp/cas-overlay/settings.gradle /cas-overlay/ && \
    cp /tmp/cas-overlay/build.gradle /cas-overlay/ && \
    cp /tmp/cas-overlay/gradle.properties /cas-overlay/ && \
    cp /tmp/cas-overlay/lombok.config /cas-overlay/

# Configurações do Gradle
RUN mkdir -p /root/.gradle && \
    echo "org.gradle.daemon=false" >> /root/.gradle/gradle.properties && \
    echo "org.gradle.configureondemand=true" >> /root/.gradle/gradle.properties && \
    cd /cas-overlay && \
    chmod 750 ./gradlew && \
    ./gradlew --version

# Sobrescreve build.gradle se necessário
COPY ./build.gradle /cas-overlay/

# Compila a aplicação CAS
RUN cd /cas-overlay && \
    ./gradlew clean build $EXT_BUILD_COMMANDS --parallel --no-daemon $EXT_BUILD_OPTIONS

# Imagem final
FROM ${BASE_IMAGE} AS cas

LABEL "Organization"="Apereo"
LABEL "Description"="Apereo CAS"

# Prepara diretórios
RUN mkdir -p /etc/cas/config /etc/cas/services /etc/cas/saml /cas-overlay

# Copia o WAR gerado na imagem de build
COPY --from=overlay /cas-overlay/build/libs/cas.war /cas-overlay/

# Expõe portas padrão do CAS
EXPOSE 8080 8443

# Ajusta PATH e diretório de trabalho
ENV PATH=$PATH:$JAVA_HOME/bin
WORKDIR /cas-overlay

# Executa o CAS
ENTRYPOINT ["java", "-server", "-noverify", "-Xmx2048M", "-jar", "cas.war"]


sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt
