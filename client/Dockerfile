FROM python:alpine

#Install dependencies (curl and s5cmd)
RUN apk --no-cache add curl && mkdir ~/s5cmd && cd ~/s5cmd && curl -S -s -L https://github.com/peak/s5cmd/releases/download/v2.3.0/s5cmd_2.3.0_Linux-64bit.tar.gz | tar -xvz && mv s5cmd /bin/ && cd ~ && rm -rf ~/s5cmd

COPY reg-api-client /bin/reg-api-client

ENTRYPOINT ["python","/bin/reg-api-client"]
