FROM node:6-alpine

RUN apk add --no-cache tini && mkdir -p /usr/src/app

WORKDIR /usr/src/app

COPY . /usr/src/app

RUN npm install && npm cache clean --force

EXPOSE 3000

CMD [ "/sbin/tini", "--", "node", "./bin/www" ]
