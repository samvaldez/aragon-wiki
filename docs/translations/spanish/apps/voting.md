# [Voting](https://github.com/aragon/aragon-apps/tree/master/apps/voting)

_**Código fuente en Github:**_ [aragon-apps/apps/voting](https://github.com/aragon/aragon-apps/tree/master/apps/voting)

La Voting app es una entidad que ejecutará un conjunto de acciones en otras entidades si los titulares de un token en particular deciden hacerlo.

### Inicialización de la app

Se crea una instancia de la Voting app con un cierto conjunto de parámetros que no serán modificables durante la vida de la aplicación:

- Token: dirección del token MiniMe cuyos titulares tienen poder del voto proporcional a sus tenencias.
- Soporte requerido: que porcentaje de votos deben ser positivos para que se realice la votación. Hacerlo al 50% podría ser una 'democracia simple'.
- Cuórum mínimo de aceptación: porcentaje mínimo de todo el suministro de tokens que debe aprobarse para que se pueda ejecutar la votación.
- Tiempo de votación: número de segundos en que se abrirá un voto, si no se cierra prematuramente para un soporte destacado.

El único parámetro que puede ser cambiado es el 'Cuórum mínimo de aceptación' para protección, en caso de que no haya suficiente participación de los votantes.

### Nota sobre porcentajes
Si eres un usuario final, puedes saltarte esta sección, pero si eres un desarrollador y quieres manipular directamente el [Voting contract](https://github.com/aragon/aragon-apps/blob/master/apps/voting/contracts/Voting.sol) estas son algunas notas que debes considerar.

Las variables “Soporte requerido” y “Cuórum mínimo de aceptación” son porcentajes que se expresan entre cero y un máximo de 10^18 (que representa el 100%). Como consecuencia, es importante considerar que el 1% actualmente está representado por 10^16.

Además, debe pasar al contracto inteligente el número actual, no la notación científica, por tanto:
- 10^16 es 10000000000000000 (ó 1 con 16 ceros)
- 10^18 es 1000000000000000000 (ó 1 con 18 ceros)

Aquí hay algunos porcentajes que se pueden usar

Porcentaje | Notación Científica | entrada actual pasada al contracto inteligente
------------ | ------------- |  -------------
0%     | 0         | 0
1%     | 10^16         | 10000000000000000
10%   | 10*10^16   | 100000000000000000
20%   | 20*10^16   | 200000000000000000
25%   | 25*10^16   | 250000000000000000
50%   | 50*10^16   | 500000000000000000
60%   | 60*10^16   | 600000000000000000
70%   | 70*10^16   | 700000000000000000
75%   | 75*10^16   | 750000000000000000
100% | 100*10^16 | 1000000000000000000


### Ciclo de vida del voto

#### Creación
```
votingApp.newVote(bytes _executionScript, string _metadata)
```

Un nuevo voto es inicializado con:

- Script de ejecución: [EVM call script](../../documentation/aragonOS/#evm-call-script) se ejecuta al aprobar el voto, contiene una serie de direcciones y cargas útiles de datos de llamada que se ejecutarán.
- Metadatos:  una cadena arbitraria que se usa para describir la votación.

La Voting app se ajusta a la [aragonOS Forwarder interface](../../documentation/aragonOS/#forwarders). Una acción de reenvio genérica creará un voto con la secuencia de comandos de ejecución proporcionada y los metadatos vacíos.

Cuando se crea un voto, se guarda una referencia del número de bloque anterior como el bloque de instantáneas para el voto. La razón por la que se usa el número de bloque anterior es para evitar el doble voto en el mismo bloque en el que se crea el voto. Cada vez que se realiza una votación, se comprueba el MiniMeToken asociado a la aplicación para el saldo del token del votante en el bloque de instantáneas.

#### Emisión de votos
```
votingApp.vote(uint256 _voteId, bool _supports)
```

Para emitir un voto, todas estas condiciones deben cumplirse:

- El emisor tenía un saldo positivo en el balance del token en el voto del bloque de instantáneas. 
- El voto no ha expirado.
- El voto no ha sido ejecutado.

Si el voto emitido es a favor de la votación, el número de tokens en poder del emisor, en el bloque de instantáneas, se agregará al contador 'yea'. En caso de que un voto sea en contra, lo agregará al contador 'nay'.

Después de cualquier voto emitido, el contrato verifica si un voto ya cuenta con el respaldo completo para ser ejecutado (incluso si todos los demás votaron en contra, el voto aún sería aprobado), en ese caso la votación se ejecuta y se cierra.

#### Ejecución del voto
```
votingApp.executeVote(uint256 _voteId)
```

Después de que una votación haya expirado en el tiempo (y no se permiten más votos), el resultado de la votación puede ser ejecutado por cualquier persona si fue aprobado. Para que una votación se considere aprobada, ambas condiciones deben ser verdaderas:

- El porcentaje de 'yea' del número total de votos es mayor o igual que el parámetro global 'Soporte requerido'.
- El soporte 'yea' es mayor o igual que el parámetro global del 'Cuórum mínimo de aceptación'.

#### Cambio del cuórum mínimo de aceptación
```
votingApp.changeMinAcceptQuorumPct(uint256 _minAcceptQuorumPct)
```

En cualquier momento, se puede modificar el quórum mínimo de aceptación para las nuevas votaciones.

Cualquier votación abierta mantendrá el valor del cuórum mínimo de aceptación de cuando se creó.

#### Reenvío

Usando la interfaz común [Forwarding](../../documentation/aragonOS/#forwarders) ejecuta una acción `votingApp.newVote(...)`. Se verifica el ACL para ver si el emisor tiene permisos para crear un voto.
