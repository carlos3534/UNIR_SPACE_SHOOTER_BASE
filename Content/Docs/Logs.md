### Observaciones Generales del juego
El juego se muestra con movimiento, enemigos y también parap oder hacer disparos. Los disparos tienen sonido, explosión de colisión y también se puede ver el efecto de la bala al impactar con los enemigos. Los enemigos tienen un comportamiento básico, se mueven hacia el jugador y pueden ser eliminados con los disparos. A continuación observaciones más detalladas:

- Existen 2 tipos de enemigos, con 2 movimientos diferentes
- No existe sonido de background, sería bueno tenerlo
- El sistema de puntuación se basa SOLO en el progreso del jugador. Matar a un enemigo no me da puntos
- La bala atravise a todos los enemigos en linea, sería bueno que esto sea un poder diferente que se acabe y que la bala sea 'nerfeada'
- No existe colisión hasta el jugador, no encontré forma de perder dentro del juego.
  - Existe colisión pero noafecta al personaje y los enemigos se destruyen a él mismo
- No se presencia sonido de colisión entre nave enemiga y el jugador.

#### Commit 1
+ Se agrega este log y música de ambiente al juego

#### Commit 2
+ Se arregla para que la colisión sea sólo con las balas (no con el jugador)
+ Se agrega daño al jugador por enemigo y ammo
+ Se agrega tiempo de invulnerabilidad al jugador después de recibir daño
+ Se arregla para que las balas se destruyan en colisión
+ Las balas del player ya evita una segunda colisión luego de golpear un enemigo

#### Commit 3
+ Delay de disparos corregido, ahora hay un random sin rango (que sería bueno poner) de los disparos
+ Disparos hacen Beep
+ Se agregó un conteo de enemigos eliminados y se muestra en pantalla

#### Commit 4
Se verifica que no hay feedback al recibir un golpe, la vida baja pero no hay feeling al respecto.
Ideas a crear:
- Efecto de parpadeo al recibir daño
- Es necesario crear lógica de crear un item cuando el enemigo se destruya, esta lógica debe considerar la vida del jugador.
- Barrera o látigo que permita al jugador destruir balas en un radio, de forma que pueda moverse de algún punto a otro.
  - Esta barrera podría recargarse con alguna barra de energía que se vaya llenando con el tiempo o con enemigos eliminados.
  - Se podría tener cooldown de al menos dos habilidades
    - Estas habiliades serían un uso? o se reemplazaría si se encuentra alguna otra barrera?
    - En este sentido, si se coge el mismo subiría de nivel la habilidad?

+ Creado el Pack de vida para curarse

#### Commit 5
+ Se agrega lógica de que aparezca el health item
+ Se agrega efecto y colisión del paquete de vida
+ Implementado el paquete de vida, al recogerlo se suma vida al jugador
+ Mejorado la iluminación... creo

#### Commit 6
+ Nuevo enemigo agregado, enemigo de tipo Cohete/Kamikaze, aparece y se dirige hacia el jugador, al colisionar con él hace daño y se destruye.
+ Enemigo agregado genera daño, es destruible y se destruye al final también.

#### Commit 7
+ Se trabaja con lógica del nivel, las naves rojas son más rápidas a medida que sube el nivel
+ El Cohete sube de nivel a ser nivel 3, llega más rápido