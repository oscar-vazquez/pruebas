# Proceso Rehabilitaciones MAC

> **Nota**: tener en cuenta al momento de la puesta en marcha, que se deben procesar en MAC todas las ordenes pendientes 
en SOLDIME antes de comenzar a trabajar con eOrder.

Buscar todas las ordenes de **rehabilitación** que aún no fueron exportadas a eOrder.

~~~ sql
SELECT nro_servicio,
       fecha_solicitud,
       numero_cliente
  FROM servicio_cab 
 WHERE id_servicio = 2 
   AND estado_transmision = 'T' 
 ORDER BY fecha_solicitud
~~~

Por cada orden encontrada en la consulta anterior, buscar cortes previos para el cliente.
Seleccionamos el último, el estado del mismo y su resultado.

~~~
SELECT FIRST 1 
       a.nro_servicio,
       a.fecha_solicitud,
       a.estado_trasmision,
       b.sit_encon,
       b.accion
  FROM servicio_cab a
       JOIN servicio_corte b ON (b.nro_servicio = a.nro_servicio)
 WHERE a.id_servicio = 1 
   AND a.numero_cliente = <servicio_cab.numero_cliente para la Reposición> 
 ORDER BY a.fecha_solicitud DESC
~~~

#### No existe corte previo
Esta situación no es normal que ocurra, para que exista una rehabilitación debe haber un corte previo. 
Pero se podría dar esta situación si se depuraron las tablas de la interface.

> **Acción**: Pasar la rehabilitación a eOrder

#### Existe corte previo
De acuerdo al valor que tenga el campo _estado_transmision_ se deben tomas distintas acciones.

Lista de valores posibles para el campo _estado_transmision_.

| Estado | Descripción |
|--------|-------------| 
| T | A Trasmitir |
| W | En eOrder |
| R | Resultado Ingresado |
| N | Cancelado del lado de la interface <sup><a href="#ref-1">1</a></sup>|
| E | Eliminada |
| M | Resultado ingresado en MAC |

<a name="ref-1">(1)</a> xxxxxxxxxx

 
Los dos últimos estados son utilizados por el MAC y no por la interface con el sistema gestor de ordenes de trabajo (SOLDIME / eOrder).


La secuemcia de estados para un ciclo normal es:
1. La orden se inserta en las tablas de interface con el estado  **T**
2. La interface SOLDIME/eOrder toma la orden y le cambia el estado **W**
3. La interface SOLDIME/eOrder ingresa el resultado del trabajo en campo cambiando el estado a **R**
4. Cuando se pasa el resultado a las tablas del MAC se cambia a estado **M**

Las ordenes en estado **N** cuando se pasan a las tablas del MAC se le cambia el estado a **E**

A continuación el análisis de de las acciones a tomar según el estado del corte.

##### Estado T
Primero se pasan los cortes y luego las rehabilitaciones por lo cual no se debería estar procesando un rehabilitación 
para un corte que no haya sido pasado a eOrder.

Esta situación se toma como un error a ser reportado.

##### Estado W
El corte se encuentra en SOLDIME/eOrder
 
##### Estado R
##### Estado N
##### Estado E
##### Estado M



* Si no existe corte previo registrado, el corte previo está notificado como cancelado (estado = 'N') o como finalizado en terreno pero no efectivo (estado = 'R', sit_encon = 'NR' y accion categorizada como CORTE NO EFECTUADO - ver tabla de SOLDIME): marcamos a la reposición en MAC como no realizada y así finalizamos el procesamiento de dicha orden de reposición.

~~~
UPDATE servicio_cab 
   SET estado = 'C' 
 WHERE nro_servicio = <nro. de servicio de la repo> 
~~~

* Si el corte previo que está notificado como efectuado en terreno (estado = 'R') y la accion corresponde a la categoría de CORTE EFECTUADO (ver en tabla de SOLDIME de acciones realizadas), entonces se envía a eOrder la orden de reposición (documento [InterfazCreacionTdC_Repo_MAC.md](InterfazCreacionTdC_Repo_MAC.md)).

* Si el corte previo está notificado como pendiente de transmitir a eOrder (estado = 'T'): marcamos tanto al corte como a la reposición en MAC como no realizadas y así finalizamos el procesamiento de dicha orden de reposición.

~~~
UPDATE servicio_cab 
   SET estado = 'C' 
 WHERE nro_servicio IN (<nro. de servicio del corte>, <nro. de servicio de la repo> )
~~~

* Si el corte previo está notificado como transmitido a eOrder (estado = 'W', en proceso), se debe enviar una orden de anulación del mismo a eOrder.

Para anular el corte se debe utilizar el WS de eOrder: **P005_PeticionSuspension_ARG.wsdl**, con los siguientes valores:

>> Nota 1: Formato del código externo de Orden: M########### (para MAC: son 14 caracteres en total, comienzan con el caracter M y luego el número de servicio completado con hasta 13 ceros a izquierda). 

| Elemento | Valor |
| --------- | --------- | 
| LLAVE_SECRETA | -*Definir*- |
| CODIGO_DISTRIBUIDORA | ESU |
| CODIGO_SISTEMA_EXTERNO_DE_ORIGEN | ESUMAC |
| CODIGO_TIPO_DE_TDC | SCR.02 |
| CODIGO_EXTERNO_DEL_TDC | servicio_cab.nro_servicio con formato (ver Nota 1) |
| TIPO_OPERACION| -*Averiguar*-  |

Evaluar el resultado que retorna el WS: 

* Si el Código de Error/Resultado es "00" ("Operación ejecutada con éxito"): marcamos tanto al corte como a la reposición en MAC como no realizadas y así finalizamos el procesamiento de dicha orden de reposición.

~~~
UPDATE servicio_cab 
   SET estado = 'C' 
 WHERE nro_servicio IN (<nro. de servicio del corte>, <nro. de servicio de la repo> )
~~~

* Si el Código es "03" ("Operación fallida porque el TdC ya está en ejecución o se ha todavía manejado"): eOrder no pudo cancelar el corte en forma inmediata. En este caso, en los sistemas locales, el corte sigue en proceso y la reposición sigue en estado pendiente hasta que por otras interfaces en forma asincrónica llegue alguna novedad del corte.

* En otro caso: error no manejable. Registrar el evento (enviar mail) pero no actualizar tablas en MAC.







considerar el momento del cambio
que hacer con las ordenes
debe quedar todo cerrado
los cortes los cancelo diariamente
las repos quedan pendientes si no se realizaron

al momento de cambiar de sistema se debería correr el cierre diario para finalizar los cortes
cambiar el estado de todas las repos para que se vuelvan a trasmitir
generar el nuevo lote de cortes
