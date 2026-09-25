# TransferCitizen — Sincronización entre Operadores

> **Fuente:** Comunicación del profesor / acuerdo de interoperabilidad entre equipos.
>
> Este documento contiene especificaciones comunicadas por el profesor relacionadas con la transferencia de ciudadanos entre operadores. No corresponde a una decisión arquitectónica interna del equipo.

TransferCitizen
 
Sincronización entre Operadores
General
Buenas tardes muchachos.
Espero estén bien, y les esté rindiendo el trabajo.
Les escribo porque, a partir de conversaciones con algunos compañeros de varios equipos, identificamos la necesidad de sincronizarnos con relación a la transferencia de ciudadanos (tanto recepción como envío).
Sabemos que cuando un ciudadano desea transferirse a otro operador, nosotros, aparte de todas esas operaciones para obtener la info del ciudadano, debemos seguir el flujo a continuación:
Decirle a gov carpeta que borre ese usuario.
Consumir el API del otro operador (http://[your-context]/api/transferCitizen).
Esperar la respuesta del otro operador.
Borrar la info (tanto de BD como del bucket) cuando el otro operador confirme.
 
Ese paso 3 es crucial para evitar que ocurran problemas al momento de que el otro operador esté descargando y cargando los archivos a su bucket.
Por eso, propongo dos cosas:
Que hagamos uso de una de las APIs que el profe dejó en el enlace que nos compartió. Esa API sería la de confirmación y sería “http://[your-context]/api/transferCitizenConfirm”.
Que añadamos esa API de confirmación como parte del body de la API de transferCitizen, con el fin de que el operador pueda saber a qué API llamar.
Así, tendríamos que el body de la API de transferCitizen sería:
 
{
    "id": 1032236578,
    "citizenName": "Carlos Castro",
    "citizenEmail":"myemail@example.com",
    "urlDocuments" : {
    "URL1": ["http://example.com/document1"],
    "URL2": ["http://example.com/document2"]
    },
    "confirmAPI": "http://[your-context]/api/transferCitizenConfirm"
}
 
Y el body de la API transferCitizenConfirm sería el siguiente:
 
{
     “id”: 1032236578,
     “req_status”: 1
}
 
Donde “id” correspondería al id del usuario/ciudadano que acaba de recibir, y que ya el operador que originalmente lo envió puede eliminar completamente.
Y, “req_status” sería un mensaje que indique el estado de la request, el cual tendría dos valores por cuestiones de simplicidad: 1 para éxito, 0 para fracaso.
De esta forma, en el flujo de la recepción de un usuario, habrá un paso en el cual se deberá hacer el llamado a la API de confirmación para indicarle al otro operador que recibió la información correctamente, y que ya terminó de subir los archivos/documentos del usuario, con el fin de que ese pueda proceder con toda la eliminación correspondiente.
Me disculpan muchachos por el mensaje tan largo, pero considero que es bien importante que, para éxito de todos, estemos en la misma página respecto a este asunto.
Me cuentan si están de acuerdo o si proponen alguna otra solución para sincronizarnos.
Muchas gracias muchachos, y Feliz tarde!
 