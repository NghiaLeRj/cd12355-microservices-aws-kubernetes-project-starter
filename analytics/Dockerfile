FROM python:3.8
COPY ./ /app
WORKDIR /app

RUN pip install -r $PWD/requirements.txt

EXPOSE 5153
EXPOSE 5433

#Export variable
ENV DB_USERNAME nghiale
ENV DB_PASSWORD $POSTGRESQL_PW
ENV DB_HOST host.docker.internal
ENV DB_PORT 5433
ENV DB_NAME mydatabase

#Run application
CMD [ "python", "app.py"]

#Command run
#docker run -it  -p 5433:5433 -p 5153:5153 -e POSTGRESQL_PW=<password> test-coworking-analytics