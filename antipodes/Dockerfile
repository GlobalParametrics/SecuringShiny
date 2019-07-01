FROM rocker/r-ver:3.6.0

RUN apt-get update -y
RUN apt-get install libpng-dev -y
RUN R -e "install.packages('remotes')"

RUN R -e "remotes::install_github('GlobalParametrics/antipodes')"

EXPOSE 3838

CMD R -e 'antipodes::launchApp()'
