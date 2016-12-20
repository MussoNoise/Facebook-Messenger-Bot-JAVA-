**Guida alla creazione di un Facebook Bot usando JAVA 7 e il Google App Engine**

**Obiettivo:**

L’obiettivo di questa guida è quello di creare un bot per Facebook Messenger in Java. Questo Bot sarà supportato da funzioni di NLP (natural Language programming) grazie ad [API.ai](https://console.api.ai/api-client/).
**Strumenti utilizzati: **

1.Ecpliple Mars 2.0
2.Plug-in di Google App engine per Eclipse

**Prerequisiti:**
1. Un’account developer Facebook (https://developers.facebook.com).Puoi semplicemente loggare con le tue credenziali Facebook.
2. Una pagina Facebook
3. Conoscenze di Java ed in particolare su come caricare un’applicazione Java su un dominio Google usando il Google App Engine plug-in per Eclipse. 

**Funzionamento generale:**

L’idea di fondo è semplice, quando un’utente invia un messaggio alla pagina Facebook, i server di Facebook inviano una richiesta POST ad un indirizzo url deciso da noi che ci consente di gestire questo “evento”, ed eventualmente di rispondere alla richiesta inoltrata da Facebook. Infatti, lo scopo del nostro codice Java che andremo a caricare sulla pagina URL citata in precedenza, chiamata generalmente [Webhook](https://it.wikipedia.org/wiki/Webhook), è quello di gestire la richiesta proveniente di da facebook, estrapolando il testo inviato dall’utente iniziale e il suo personal ID,in modo da poter inviargli una risposta testuale utilizzando la Facebook Send API ([https://developers.facebook.com/docs/messenger-platform/send-api-reference](https://developers.facebook.com/docs/messenger-platform/send-api-reference)).

**Setup Facebook App:**

[(https://developers.facebook.com/docs/messenger-platform/guides/quick-start]((https://developers.facebook.com/docs/messenger-platform/guides/quick-start))
La guida ufficiale di facebook è ben dettagliata e spiega esattamente come creare un’applicazione Facebook. Il problema è che il linguaggio di programmazione usato in tutte le guide ufficiali è il Node.js. L’obiettivo di questa guida è quello di creare un Bot di Facebook Messenger interamente in Java, quindi vediamo passo passo come configurare al meglio l’applicazione:
Innanzitutto crea un’applicazione Facebook e una pagina Facebook. Nella Dashboard della tua applicazione clicca su “Aggiungi prodotto” e abilita Messenger.

![alt tag](https://raw.githubusercontent.com/MussoNoise/Facebook-Messenger-Bot-JAVA-/master/Img/AbilitaMSN.png)

**Configurazione Webhook:**

Facebook compie una verifica sul nostro webhook per essere certo che la pagina chiamata sia effettivamente di nostra proprietà e che accetti chiamate sicure Https. Infatti, Facebook valida solamente Webhook dotati di certicati SSL validi. Questo problema è ovviato da servizi gratuiti di Hosting come Google App Engine che mettono a disposizione domini Https con certificati che coprono tutti i sottodomini del sito caricato. La verifica consiste in una chiamata da parte dei Server Facebook di tipo GET con questa forma:

[https://%WebHookUrl%/?hub.verify_token=verifytoken&hub.challenge=randomCode](https://%WebHookUrl%/?hub.verify_token=verifytoken&hub.challenge=randomCode)

Se il servizio chiamato risponde con il valore hub.challenge effettivamente inviato in precedenza, il webhook è validato. In java questa operazione può essere svolta in questo modo:
Apri Eclipse,crea un nuovo progetto “web application project” . 
Ricorda di inserire il tuo project ID, puoi comunque farlo in un secondo momento. Lasciando la spunta su”generate project sample code” Eclipse genererà automaticamente una Servlet chiamata %projectName%Servlet.java; a questo punto sostituisci il metodo doGet al suo interno con questo:
<details> 
  <summary>Mostra Codice:</summary>

     public void doGet(HttpServletRequest req,HttpServletResponse res) throws IOException{
			 String queryString = req.getQueryString();
			 String msg = "";
			if (queryString != null) {
				String verifyToken = req.getParameter("hub.verify_token");
				String challenge = req.getParameter("hub.challenge");
				msg = challenge;
			
	           if(verifyToken.equals("verify")){
			res.getWriter().write(msg);
			res.getWriter().flush();
			res.getWriter().close();
			res.setStatus(HttpServletResponse.SC_OK);
			return;
	           }
			}
		}
        </details>
	
Il codice svolge esattamente le operazioni descritte in precedenza, in caso di chiamata Get, viene acquisito il testo della richiesta,e nel caso che non sia nulla,procede salvando i parametri passati nella richiesta get.Il secondo controllo serve per verificare che il verifytoken impostato su facebook (vedi paragrafo successivo) sia uguale a “verify”.Se tutti i controlli vanno a buon fine il nostro webhook risponde con il valore di hub.challenge.
A questo punto dobbiamo effettivamente comunicare a Facebook l’URL del nostro webhook:
Nella dashboard della tua applicazione clicca su “configura Webhook”:

![alt tag](https://raw.githubusercontent.com/MussoNoise/Facebook-Messenger-Bot-JAVA-/master/Img/SetHook.png)

L’url di Callback è l’indirizzo in cui abbiamo caricato il nostro codice, nel nostro caso sarà l’indirizzo della Servlet sopra descritta.
Per il nostro progetto basta spuntare “messages” “messaging_postback” “message_reads”.
Se tutto va buon fine comparirà una schermata di questo tipo:

![alt tag](https://raw.githubusercontent.com/MussoNoise/Facebook-Messenger-Bot-JAVA-/master/Img/HookSetted.png)
N.B.
Perché ho usato 1-dot-facebottest88.appspot.com/… ??
Se inseriamo il dominio completo www.facebottest88.appspot.com la validazione non andrà mai a buon fine, questo perché Facebook non riesce a leggere il certificato SSL di questo dominio. Invece, inserendo 1-dot- funzionerà tutto correttamente, dopo aver aggiunto questo codice nel file web.xml del nostro progetto:

<details> 
  <summary>Mostra Codice:</summary>
    <security-constraint>
    <web-resource-collection>
        <web-resource-name>everything</web-resource-name>
        <url-pattern>/*</url-pattern>
    </web-resource-collection>
    <user-data-constraint>
        <transport-guarantee>CONFIDENTIAL</transport-guarantee>
    </user-data-constraint>
    </security-constraint>
</details>

Questo accade solo con le autenticazioni di Facebook,infatti altri servizi famosi come Telegram riescono a verificare immediatamente domini classici www.%nomeSito%.appspot.com

Fonte:
https://cloud.google.com/appengine/docs/java/config/webxml#Secure_URLs

Adesso nel menù di impostazione di messenger comunichiamo a Facebook la pagina pubblica che sarà legata al nostro webhook:

![alt tag](https://raw.githubusercontent.com/MussoNoise/Facebook-Messenger-Bot-JAVA-/master/Img/Foto%20Lega%20pagina.png)
 
L’applicazione è ora correttamente impostata Facebook side,ma effettivamente il nostro Bot non risponde ancora nulla quando qualcuno invia un messaggio alla pagina FB.

**Gestione Eventi Provenienti da FB:**

Quando un utente invia un messaggio alla nostra pagina FB, facebook crea un’ “Evento”.Questo Evento può essere di vario tipo, noi in questa guida ci soffermiamo solamente sulla gestione di Eventi testuali.  Vediamo dunque come funziona tecnicamente come gestire questi eventi: Se l’utente “F.Mussini” invia “prova” alla nostra pagina FB, la pagina a sua volta compie una chiamata POST passando come body un messaggio testuale serializzato in [JSON](https://it.wikipedia.org/wiki/JavaScript_Object_Notation) di questo tipo:

{object:page,entry:[{id:967683980042454,time:1480516467930,messaging:[{sender:{id:1212697882118340},recipient:{id:967683980042454},timestamp:1480516467899,message:{mid:mid.1480516467899:d81e516520,seq:2042,text:prova}}]}]}	

Se estrapoliamo i campi “text” e “id” saremo in grado di eleborare una risposta in base al testo inviato dall’utente e di rinviarlo indietro all’id utente corretto. Ci sono vari modi per dividere un JSON, comprese [librerie java apposite](http://codingjam.it/gson-da-java-a-json-e-viceversa-primi-passi/). Personalmente per includere meno librerie esterne possibili ho semplicemente diviso il messaggio JSON usando semplici funzioni (Split,substring etc..). Salvare il body di una chiamata POST come stringa non è immediato, infatti dobbiamo convertire il flusso di byte in Stringa:

***CODE ***  (By StackOverflow)


    public String StreamToString(final InputStream is, final int bufferSize) {
    //Trasforma il flusso di byte proveniente dallo Stream in Stringa
	    final char[] buffer = new char[bufferSize];
	    final StringBuilder out = new StringBuilder();
	    try (Reader in = new InputStreamReader(is, "UTF-8")) {
	        for (;;) {
	            int rsz = in.read(buffer, 0, buffer.length);
	            if (rsz < 0)
	                break;
	            out.append(buffer, 0, rsz);
	        }
	    }
	    catch (UnsupportedEncodingException ex) {
	        /* ... */
	    }
	    catch (IOException ex) {
	        /* ... */
	    }
	    return out.toString();
	}
	

    
Il nostro programma a questo punto avrà quindi questa macrostruttura:

    Metodo doGet{   // Se arriva una richiesta di validazione webHook
    ..
    }
    Metodo doPost{  //Se arrivare un POST da FB
    Chiamata a StreamToString per salvare il JSON della richiesta;     
    
    Chiamata ad un metodo per salvare il testo inviato dall’utente nel JSON;  
    
    Chiamata ad un metodo per salvare l’id del mittente;                                                     
    }  
    
Scarica il programma completo per guardare nel dettaglio questi metodi commentati.   

**USO DELLA FACEBOOK SEND API:**	([Documentazione Ufficiale](https://developers.facebook.com/docs/messenger-platform/send-api-reference))

Per inviare un messaggio ad un utente dobbiamo effettuare una chiamata POST a https://graph.facebook.com/v2.6/me/messages?access_token=%YourPAGEtoken% ;                        

Il token è un codice generato casualmente da FB ed associato unicamente alla tua pagina. Il tuo personale Token lo puoi ritrovare nella Dashboard dell’applicazione Facebook.

Settando come Header :
Content-Type: "application/json;charset=utf-8”   

e passando come body una Stringa serializzata sempre in JSON del tipo: 

{"recipient":{"id:IdMittente"},"message":{"text":"Hei sono vivo"}}

In java una chiamata POST usando solo java.net e java.io ha questa forma:               

*** Code: ***

    URL url = new URL(“Url destinazione”);
    HttpURLConnection connection = (HttpURLConnection) url.openConnection();
    connection.setConnectTimeout(5000);
    connection.setReadTimeout(5000);connection.setRequestMethod("POST");
    connection.setDoOutput(true);
    connection.setRequestProperty("ContentType","application/json;charset=utf-8");  
    OutputStreamWriter out = new OutputStreamWriter(connection.getOutputStream(),"UTF-8");
    out.write("Testo da inviare”);
    out.flush();  
    out.close();   		 
    connection.getResponseCode();}
    connection.disconnect();	
	    }
        
**Echo BOT:**

A questo punto abbiamo tutte le informazione per scrivere il nostro primo Bot completamente funzionante. Se volessimo ad esempio creare un Bot che semplicemente risponde all’utente con il testo ricevuto,potremmo fare in questo modo:

    doGet{ 
    ..  //metodo per l’autenticazione del webhook  (Presentato nella sezione webhook)
     }
    doPost{
    Chiamata a StreamToString per salvare il JSON della richiesta;  //presentato prima  
    Chiamata ad un metodo per salvare il testo inviato dall’utente nel JSON;  
    Chiamata ad un metodo per salvare l’id del mittente;                                                     
    }  
<details> 
  <summary>Codice Completo:</summary>
   
       @SuppressWarnings("serial")
        public class FacebookBotServlet extends HttpServlet {
    	private String token="YourToken";
	    private String TargetUrl="https://graph.facebook.com/v2.6/me/messages?access_token="+token;
	
	     public void doGet(HttpServletRequest req,HttpServletResponse res) throws IOException{
			String queryString = req.getQueryString();
			String msg = "";
			if (queryString != null) {
				String verifyToken = req.getParameter("hub.verify_token");
				String challenge = req.getParameter("hub.challenge");
				msg = challenge;
			
	           if(verifyToken.equals("verify")){
			res.getWriter().write(msg);
			res.getWriter().flush();
			res.getWriter().close();
			res.setStatus(HttpServletResponse.SC_OK);
			return;
	           }
			}
		}
	
	public void doPost(HttpServletRequest req, HttpServletResponse resp) throws IOException {
       String JSON=StreamToString(req.getInputStream(), 1000).replaceAll("\"", "");
		 URL url = new URL(getTargetUrl());
		 HttpURLConnection connection = (HttpURLConnection) url.openConnection();
		 connection.setConnectTimeout(5000);
		 connection.setReadTimeout(5000); 
		 connection.setRequestMethod("POST");
		 connection.setDoOutput(true);
		 connection.setRequestProperty("Content-Type", "application/json;charset=utf-8");  
		 OutputStreamWriter out = new OutputStreamWriter(connection.getOutputStream(),"UTF-8");  
		 if(JSON.contains("message")){ 
		 String finalText=getText(JSON);
		 out.write("{\"recipient\":{\"id\":\""+getSender(JSON)+"\"},\"message\":                   {\"text\":\""+finalText+"\"}}");
		 out.flush(); 
		 out.close();  
		 int res = connection.getResponseCode();
		 System.out.println(res); 
		 }
		 connection.disconnect();
		
	    }
	    public String StreamToString(final InputStream is, final int bufferSize) {
	    final char[] buffer = new char[bufferSize];
	    final StringBuilder out = new StringBuilder();
	    try (Reader in = new InputStreamReader(is, "UTF-8")) {
	        for (;;) {
	            int rsz = in.read(buffer, 0, buffer.length);
	            if (rsz < 0)
	                break;
	            out.append(buffer, 0, rsz);
	        }
	    }
	    catch (UnsupportedEncodingException ex) {
	        /* ... */
	    }
	    catch (IOException ex) {
	        /* ... */
	    }
	    return out.toString();
	}
	public String getTargetUrl(){
	    	return TargetUrl;
	    }
	 
    public String getText(String JSON){
	    	String [] tmp=JSON.split(",text:");
	    	String text=tmp[1].substring(0, tmp[1].indexOf("}}"));
	    	
	    	return text;
	    }
	 public String getSender(String JSON){
	    	String [] tmp=JSON.split(":");
	    	String Sender=tmp[7].substring(0, 16);
	    	return Sender;
	    }
}
</details>
