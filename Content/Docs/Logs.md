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