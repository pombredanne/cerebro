https://github.com/helm/helm/tree/master/docs
https://github.com/helm/helm/blob/master/docs/related.md
  info interesante, plugins, additional tools, etc

Package manager para Kubernetes.

El cli (helm) habla con un servidor que tenemos que tener desplegado en kubernetes (tiller).
Tiller habla con las APIs de Kubernetes y almacena información sobre el despliegue en ConfigMaps.

Es una manera de empaquetar los .yaml que usamos para desplegar una aplicación en un "paquete".
El cli nos permitirá desplegar este paquete varias veces, con distintas parametrizaciones.

https://github.com/helm/helm/blob/dev-v3/docs/charts.md#the-chart-file-structure
Internamente funciona un poco como un rol de ansible (aunque funcionan solos, no hace falta tener un "playbook"). El mapeo de nombres sería algo tipo:
 - role -> chart
 - manifest -> Chart.yml
 - defaults/main.yml -> values.yml
 - templates/tasks -> templates/
   - existe un NOTES.txt que se mostrará tras desplegar el helm
 - pasar parámetros: -e foo=far -> --set foo=bar
 - playbook.yml -> Helmfile (es un plugin https://github.com/roboll/helmfile)
 - charts/ otros charts de los que este depende
 - requirements.yaml: fichero con las dependencias de nuestro chart

En templates/ tendremos los ficheros .yaml que usaríamos con "kubectl create", definidos con el templating de go (https://golang.org/pkg/text/template/ + http://masterminds.github.io/sprig/ + https://github.com/helm/helm/blob/master/docs/charts_tips_and_tricks.md)
Mustache tomará los valores de "values.yml" y de lo que pasemos con --set

Hay repositorios (como sería ansible-galaxy), para bajarnos ya charts listos para desplegar.
helm search xxx
helm fetch foo/bar
  nos baja un .tgz con el chart

Cuando desplegamos un chart de helm, se considera una release, que tendrá su propio nombre.
El concepto es que podemos desplegar el mismo chart varias veces, cada una con su nombre y en una versión determinada.
Luego será sencillo upgradearlas o hacerles rollback.



Helm 3 tendrá un par de cambios bastante grandes:
  - no hará falta la parte servidor (tiller)
    - almacenamiento de info en custom resources
    - helm (la cli) hablará directamente con las APIs de kubernetes
  - se podrán escribir los charts en LUA
Estado de la v3: https://github.com/helm/helm/issues?utf8=%E2%9C%93&q=is%3Aissue+label%3Av3
A Oct'2018 aún no se sabe fecha para la primera alpha