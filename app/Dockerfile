FROM node:10 as build
MAINTAINER dhavaln
LABEL description="This is a multi-stage NodeJS image"
WORKDIR /src
COPY package*.json .
RUN npm install
COPY . .

FROM node:10-slim
WORKDIR /src 
COPY --from=build /src .
EXPOSE 8080
CMD ["node", "index.js"]