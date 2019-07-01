FROM rocker/r-ver:3.6.0

RUN apt-get update -y
RUN apt-get install libpng-dev wget -y
RUN R -e "install.packages('remotes')"

RUN R -e "remotes::install_github('GlobalParametrics/antipodes')"

RUN wget https://downloads.jboss.org/keycloak/6.0.1/gatekeeper/keycloak-gatekeeper-linux-amd64.tar.gz && \
  tar -xzf keycloak-gatekeeper-linux-amd64.tar.gz && \
  rm keycloak-gatekeeper-linux-amd64.tar.gz

RUN echo './keycloak-gatekeeper --config /gatekeeper.conf & R -e "shiny::runApp(system.file(\"antipodes\", package = \"antipodes\"), launch.browser = FALSE, port = 3838, host = \"127.0.0.1\")"' > startup.sh

ENTRYPOINT ["sh", "startup.sh"]
