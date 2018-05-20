# [Finance](https://github.com/aragon/aragon-apps/tree/master/apps/finance)

_**Código fuente en Github:**_ [aragon-apps/apps/finance](https://github.com/aragon/aragon-apps/tree/master/apps/finance)

El propósito de la Finance app es ser el punto central para realizar un seguimiento de los ingresos y gastos en una organización, así como realizar pagos.

La Finance app es multi-token (más ether). Con el fin de eliminar la necesidad de contar con un feed de precios de confianza (oracle) para las tazas de cambio del token, cada token se contabiliza por sí mismo para el presupuesto y los estados financieros del período.

### Definiciones

#### Período contable

Los períodos contables son el equivalente a los trimestres financieros en las organizaciones tradicionales. Su longitud se puede establecer dependiendo de cómo la organizacion quiera usarlos. 

Para cada período contable, se mantiene un saldo (llamado token statement) para cada token transaccionado, este saldo será negativo si se gastó más de lo necesitado, y positivo si no.

#### Transacciones

Una transacción registra un evento en el cual los activos fueron depositados o gastados a través de la Finance app. Las transacciones de gastos se llaman 'outgoing transactions' y los depósitos 'incoming transactions'.  Cada transacción ocurre en un período contable y cuenta hacia su estado de token.

Las transacciones no se pueden crear directamente, son registradas como el resultado de una operación (ya sea un pago ejecutado o un depósito realizado).

#### Pagos

Los pagos son la construcción utilizada para gastar usando la Finance app. Se puede crear un pago para que suceda una sola vez o puede ser un pago recurrente que se realizará de acuerdo a una programación determinada.

Si se ejecuta correctamente un pago, se crea un outgoing transaction con una referencia al pago.

#### Presupuestos

Los presupuestos permiten limitar las unidades de un token que se pueden gastar por período contable. Los presupuestos se establecen en base a cada token. Por defecto, ningún token tiene un presupuesto, lo que significa que se permite un gasto ilimitado.

Una vez se ha establecido un presupuesto para un token, la Finance app solo permitirá que la cantidad presupuestada de tokens se gaste para ese período.

### Inicialización

Para iniciar una Finance app, se requieren los siguientes parámetros:

- **Vault**: referencia a una instancia [Vault](vault.md) que la Finance app utilizará para depositar y gastar tokens. Para que funcione correctamente, la Finance app **debe tener permisos para transferir los tokens de la Vault**.
- **Ether token**: dirección de la instancia EtherToken utilizada como ether.
- **Accounting period duration**: la duración inicial de los períodos contables. Puede ser cambiado posteriormente para futuros períodos.

### Ciclo de vida

#### Depósitos

Se admiten dos mecanismos de depósito:

##### ERC20
```
token.approve(finance, amount)
finance.deposit(token, amount, string reference)
```

Después de hacer una aprobación ERC20 con la Finance app como gastador solicitando `deposit(...)` creará una transacción entrante guardando el string de referencia.

##### ERC677
```
token.transferAndCall(finance, amount, string reference)
```

Realizando un ERC677 `transferAndCall(...)` a la Finance app también activara un depósito (interceptado con el `tokenFallback`). Los datos pasados como contenido al transferAndCall se usan directamente como referencia para el depósito.

Dado que la implementación de EtherToken del aragonOS' se ajusta al ERC677, depositar Ether a la Finance app se puede realizar haciendo referencia al ether usando `etherToken.wrap()` y luego haciendo un `transferAndCall(...)` o usando un acceso rápido `wrapAndCall(...)` que realiza ambas acciones.

Esta sección está sujeta a cambios según el [ERC677 discussion](https://github.com/ethereum/EIPs/issues/677) evoluciona.

### Pagos

Dependiendo los parámetros con los que se crea un pago, puede ser un pago instantáneo por única vez, un pago recurrente para la nómina o un pago programado.

#### Crear pago
```
finance.newPayment(...)
```

Se crea un pago con los siguientes parámetros:

- **Token**: dirección del token para pago.
- **Receiver**: dirección que recibirá el pago.
- **Amount**: unidades del token que se pagan cada vez que vence el pago.
- **Initial payment time**: marca de tiempo para cuando se realiza el primer pago.
- **Interval**: cantidad de segundos que debe pasar entre las transacciones de pago.
- **Maximum repeats**: instancias máximas que se puede ejecutar un pago.

En el caso de que un pago ya se pueda ejecutar en la creación, se ejecutará.

Si se crea un pago que no se repetirá nunca más y que ya se ejecutó, sólo se registrará una transacción saliente para guardar el almacenamiento.

Un pago puede tener un anterior **initial payment time**, lo que podría ocasionar que muchas instancias del pago periódico se ejecuten en el momento de la creación del pago.

#### Ejecutar el pago
```
finance.executePayment(uint256 _paymentId)
finance.receiverExecutePayment(uint256 _paymentId)
```

La ejecución del pago transferirá al destinatario de un pago el importe correspondiente dependiendo de la hora actual. Una sola ejecución puede generar múltiples transacciones si el pago no se ha ejecutado a tiempo. Para evitar que se quede sin gas cuando se deben realizar muchas transferencias, se realiza una cantidad máxima de transferencias por ejecución (en algunos casos, se podrían necesitar múltiples ejecuciones).

Los pagos siempre pueden ser ejecutados por el destinatario, pero también existe un rol adicional en la Finance app que permite que otra entidad ejecute el pago (el destinatario obtiene los fondos en ambas instancias).

Para pagos cuyo token es el conocido EtherToken, en lugar de hacer una transferencia del token, se transferirá directamente ether al destinatario.

Una ejecución de pago puede fallar en caso de que la organización se quede sin presupuesto para el token de pago para ese período contable en particular, o la organización no tenga suficiente balance de token.

#### Inhabilitación de pagos
```
finance.setPaymentDisabled(uint256 _paymentId, bool _disabled)
```

En cualquier momento, un pago puede ser deshabilitado por una entidad autorizada. Si un pago no se ha ejecutado completamente hasta ese momento y está deshabilitado, el destinatario no podrá ejecutarlo hasta que esté habilitado de nuevo.

### Períodos

#### Ajustar duración
```
finance.setPeriodDuration(uint64 _periodDuration)
```

Establece la duración del período para los siguientes períodos. La duración del período actual no se puede modificar.

#### Establecer presupuesto
```
finance.setBudget(ERC20 _token, uint256 _amount)
```

Actualiza el presupuesto de un token para los períodos contables. El nuevo presupuesto establecido se aplica automáticamente para el período actual.

#### Eliminar presupuesto
```
finance.removeBudget(ERC20 _token)
```

Elimina el presupuesto de un token, lo que permite un gasto ilimitado de ese token en cualquier período. La eliminación del presupuesto afecta el período actual.

#### Períodos de transición
```
finance.tryTransitionAccountingPeriod(uint ttl)
```

Todas las operaciones que pueden dar como resultado la creación de una transacción (creación de un pago, ejecución de un pago, realización de un depósito) comprueban primero si el período contable necesita una transición (el período anterior ha expirado) o pueden activarse manualmente solicitando `tryTransitionAccountingPeriod(...)`.

Si pasaron muchos períodos (la última operación ocurrió hace dos períodos, pero nunca se realizó la transición), se crea un período vacío para cada uno de ellos.

Para evitar el caso en el que la Finance app no puede realizar ninguna operación porque necesita transicionar demasiados períodos contables, lo que hace que cualquier operación se quede sin gas, los períodos de transición tienen un parámetro para la cantidad de períodos en los que se realizará la transición. Las transiciones automáticas sólo realizarán 10 transiciones como máximo, y si se necesitan más, la operación fallará. En caso de que se produzca este bloqueo y los períodos no puedan hacer una transición automática, se pueden desencadenar varias operaciones de transición de períodos manualmente para eliminar el bloqueo.


### Limitaciones

- Se permite la creación de pagos, pero puede hacer que se salga del presupuesto. Puede causar una carrera para ejecutar pagos justo al comienzo de un período contable.
- Las ejecuciones de pago no caducan. Hay un vector de ataque en el que, al no ejecutar un pago en algún momento, un presupuesto puede verse afectado cuando se ejecuta. En caso de que esto suceda, el pago puede ser ejecutado por otra entidad autorizada o ser deshabilitado.
