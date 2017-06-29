## REQUISITI
- __Definizione:__ "modificare il codice in modo che, al posto di trasmettere immediatamente, provi ad "ascoltare" se qualcuno sta trasmettendo e, in caso lo trova,inizia a trasmettere anticipando il trasmettitore rilevato. Questo serve sostanzialmente a prendere il controllo di un drone che sta volando..." <br/>
_Nota:_ La trasmissione avviene in frequency hopping cambiando continuamente il canale su cui radiocomando e drone comunicano, quindi per poter rilevare l'id del radiocomando che sta già trasmettendo posso mettermi su un canale fisso ed ascoltare aspettando che il frequency hopping passi per quel canale.
- __Info aggiuntive:__
	- DSMx usa 23 canali, per mettersi in ascolto e beccare l'id si può selezionare un canale fisso su cui ascoltare e poi una volta che rilevo qualcosa 
leggo i primi 2 byte dei dati ricevuti (quando fanno il binding mandano 4 byte ma in realtà sono utilizzati solo i primi 2?) e dovrei avere l'id...
	- Con l'id genero la sequenza di canali attraverso l'algoritmo di frequency hopping esistente 
	- Ricevo pacchetto e sincronizzo la trasmissione anticipandola di un delta rispetto al normale, ovviamente poi il periodo di trasmissione resta lo stesso (11ms o 22ms), 
anticipo solo il primo pacchetto che invio.

- __Note implementazione:__ Nel caso in cui abilito la funzione INTERCEPT_RADIO, il modulo all'accensione salta tutte le operazioni di binding e resta ad ascoltare.
	Nel momento in cui trovo un altro radiocomando che trasmette, leggo il suo id e inizio direttamente a controllare il drone con quell'id	anticipando la trasmissione 
	di un delta (Xms???) [sembrerebbe che il ricevitore si sincronizza automaticamente con il primo segnale che riceve???]

## OPEN POINTS
- Approfondire come avviene la sincronizzazione (il ricevitore si sincronizza automaticamente con il primo segnale che riceve???) e come realizzare l'anticipo della trasmissione.
