# Wat is docker
Docker wordt als containers voorgesteld, maar het zijn meer jails. Het grootste verschil is dat je een container verwacht te kunnen verplaatsen van PC naar PC maar dat kan niet. Het woord komt van containerizen, wat gebruikt wordt om aan te geven dat een ruimte wordt opgedeeld in afzonderlijke ruimtes waartussen je niet kan oversteken. aka cellen bouwen.

```
docker run -it --name demo busybox sh
/ # ps ax 
docker ps
docker stop demo
Docker ps
Docker ps -a
Docker rm -v demo
```
# hoe werkt docker
Docker is gebaseerd op linux technologieen en draait ook alleen op linux. Je kan het op andere os-en draaien, maar dan draait het OS een virtuele machine met linux en daarbinnen draaien dan containers. Ze doen hun best om dat zo transparant mogelijk te maken. 

Onder linux heb je een aantal technologieen die een proces kunnen beperken in de toegang naar het host systeem, je hebt chroot, virtuele schijven op basis van mounted files, je kan virtuele netwerken aanmaken met virtuele netwerkkaarten en je kan een soort van chroot voor processen opzetten. Docker is ontstaan als stukje code wat de juiste kernel calls maakt om al die technieken ze te configureren dat je applicatie denkt als enige in het systeem te draaien. Het draait ook als PID 0 (wat weer problemen geeft soms, want PID 0 is speciaal) en als de applicatie stop “stopt de container” dit is een beetje een gek woord want er is natuurlijk geen container. Er is alleen een applicatie die van de kernel niet bij andere applicaties mag. 

# Waarom is het leuk?

## 1. single blocking process
```
docker run -it busybox sh
<Ctrl-d>
```

Dit model is enorm veel simpeler dan background processen. Vergelijk scripts met background processen vs foreground processen met programmeren met async threads of sync calls. Alleen dan met de toevoeging dat bash 0 ondersteuning heeft (while loops met sleeps die wachten op output enzo)

```
docker run -d --name duurt_lang busybox sleep 10
docker attach duurt_lang
```

## 2. process management

```
docker run -d --restart=always --name crashingbackgroundprocess  busybox sh -c 'echo foo && sleep 2'
docker logs --tail crashingbackgroundprocess
docker logs --tail crashingbackgroundprocess | wc -l
# wacht even...
docker logs --tail crashingbackgroundprocess | wc -l
```

na een keer of 10 wil hij niet meer herstarten :( Ik weet niet of dat een bug is of by design en ook niet of de bug in de mac-docker schil of echte docker zit.

## 3. meer configuratie buiten de app
In plaats van iedere app zijn eigen poortconfiguratie te laten hebben denkt de app gewoon dat hij op 80 zit en expose je wat je wil exposen

```
docker run -p 8080:80 nginx
```

## 4. filesystem snapshots
Naast de applicatie die draait houdt docker ook een filesystem bij. De grote truc is dat je die filesystems kan snapshotten en dat je op een snapshot door kan werken via een copy-on-write systeem. Van origine gebruikten ze hier AUFS voor maar er is vrij veel innovatie in dit gedeelte van docker. De lagen en de snapshots zorgen ervoor dat docker zo populair is geworden.

```
docker run -it --name firstbusybox busybox sh
touch foo
<ctrl-d>
docker run -it busybox sh
ls foo
docker start -ai firstbusybox
ls foo 
```

## 5. Het geheel is groter dan de som der delen
Als je namelijk een applicatie hebt die alleen toegang heeft tot de dingen die je expliciet voor hem configureert, en waar je alleen mee kan communiceren als je dat expliciet configureert, en je geeft hem een filesystem waar hij alleen bijkan en waar alle changes expliciet op moeten worden gecommit anders ben je ze bij de volgende run weer kwijt, dan wordt je als gebruiker opeens getriggerd om een heel kleine set van files en configuratie aan te leggen die specifiek zijn voor die applicatie.

Zie bijvoorbeeld: 
 1. https://hub.docker.com/r/neo4j/neo4j/~/dockerfile/ die verwijst naar
 1. https://hub.docker.com/_/java/ en zo kom je bij
 1. https://github.com/docker-library/openjdk/blob/89851f0abc3a83cfad5248102f379d6a0bd3951a/8-jre/Dockerfile

## Wat kan ik dan?

 - Distributie die mensen kunnen vertrouwen
 - Lokaal de serveromstandigheden reproduceren
 - pre-push hook: https://github.com/HuygensING/timbuctoo/blob/develop/timbuctoo-instancev4/pre-push
 - clean database store: commit je database changes en start steeds een nieuwe container vanaf een known image
 - autoscaler: http://kubernetes.io/docs/hellonode/
 - Snel een tooltje testen: Stel je wil is kijken of de app [upsource](https://www.jetbrains.com/upsource/) wat is:
   1. https://hub.docker.com/search/?isAutomated=0&isOfficial=0&page=1&pullCount=1&q=upsource&starCount=0
   1. `docker run -p -it 8080:8080 esycat/upsource`
   1. Speel ermee
   1. Wil je hem op een server?
     1. Vind een leuke docker cloud (ik gebruik rackspace Carina, want gratisch) 
     1. Switch naar de docker cloud instance door wat environment variables in te stellen (docker daemon is een HTTP server)
       `Source ~/Downloads/test/test.env`
   1. Zelfde commando nog een keer:
      `docker run -d --name upsource -p 82:8080 --restart=always esycat/upsource`
   1. Ga naar de url op internet! (voor de demo http://146.20.68.219:82 )
   1. In plaats van een kapotte pc heb je alles schoon en productieklaar

# Dingen om op te letten:

- Docker op de mac (gebruik de private beta want filesystem mappings werken heel slecht onder de huidige versie)
- Doe een rm -v of  docker volume ls -qf dangling=true
- Docker-compose https://docs.docker.com/compose/overview/
- Antipatterns die ik ben tegengekomen
	 - Complexe build scripts 
	 - Veel heen en weer mounten
- Een docker user is een root user! (Kun je vast wat aan doen, maar dan moet je wel snappen wat je doet)
- Als je een volume in een docker container mount en je maakt files aan, dan worden die door root aangemaakt
- Twelve factor apps: http://12factor.net/
