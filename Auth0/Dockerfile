FROM rocker/shiny

RUN apt-get install nodejs git -y

RUN git clone https://github.com/GlobalParametrics/antipodes.git && \
  R -e "install.packages(c('leaflet', 'shiny'))" && \
  R -e "install.packages('antipodes', repos = NULL, type='source')"

RUN apt-get install curl software-properties-common -y && \
  curl -sL https://deb.nodesource.com/setup_10.x | sudo bash - && \
  apt-get update -y && \
  apt-get install nodejs -y

RUN git clone https://github.com/auth0/shiny-auth0.git

WORKDIR shiny-auth0

RUN npm install

EXPOSE 3000

RUN touch env && ln -s env .env

RUN echo "shiny-server & node bin/www" > startup.sh

ENTRYPOINT ["sh", "startup.sh"]
