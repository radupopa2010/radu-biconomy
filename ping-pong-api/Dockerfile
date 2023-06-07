FROM node:20
# Add https://github.com/krallin/tini to run as PID 1 in container
RUN apt-get update && apt-get install -y tini
USER node
EXPOSE 3000
ENV PORT=3000
WORKDIR /ping-pong-api
COPY controller.js package.json routes.js server.js .
RUN npm install
ENTRYPOINT ["/usr/bin/tini", "--", "npm", "start"]