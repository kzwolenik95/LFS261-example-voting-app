FROM maven:3.9.6-sapmachine-17
WORKDIR /app
COPY . .
RUN mvn package && \
    mv target/worker-jar-with-dependencies.jar /run/worker.jar && rm -rf /app/*
CMD java -jar /run/worker.jar