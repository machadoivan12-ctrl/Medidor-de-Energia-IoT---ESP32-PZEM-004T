# Medidor-de-Energia-IoT---ESP32-PZEM-004T
Este repositório contém o código-fonte de um medidor de energia digital desenvolvido para o Trabalho de Conclusão de Curso (TCC) em Eletrotécnica na ETEC de Piraju.


#include <WiFi.h>
#include <ESPAsyncWebServer.h>
#include <PZEM004Tv30.h>

// ==========================================
// 1. COLOQUE SEUS DADOS DO WIFI AQUI
// ==========================================
const char* ssid = "NOME DA INTERNET";
const char* password = "SENHA DA INTERNET";

// Define a Serial2 do ESP32 (Pinos 16 e 17)
// Conecte: TX do PZEM no pino 16 | RX do PZEM no pino 17
PZEM004Tv30 pzem(Serial2, 16, 17);

AsyncWebServer server(80);

// Esta é a parte da "cara" do site que vai aparecer no seu celular
const char index_html[] PROGMEM = R"rawliteral(
<!DOCTYPE HTML><html>
<head>
  <meta charset="UTF-8">
  <title>Meu Medidor de Energia</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    body { font-family: Arial; text-align: center; background-color: #2c3e50; color: white; }
    .card { background: #34495e; border-radius: 10px; padding: 20px; margin: 15px; display: inline-block; width: 250px; box-shadow: 0 4px 8px rgba(0,0,0,0.3); }
    .label { font-size: 1.2rem; color: #bdc3c7; }
    .value { font-size: 2.5rem; font-weight: bold; color: #f1c40f; }
    h1 { margin-top: 50px; }
  </style>
</head>
<body>
  <h1>Monitor de Energia IoT</h1>
  <div class="card"><div class="label">Voltagem</div><div class="value"><span id="v">0</span>V</div></div>
  <div class="card"><div class="label">Corrente</div><div class="value"><span id="i">0</span>A</div></div>
  <div class="card"><div class="label">Potência</div><div class="value"><span id="p">0</span>W</div></div>
  <div class="card"><div class="label">Total (Energia)</div><div class="value"><span id="e">0</span>kWh</div></div>

  <script>
    // Função que pede os dados para o ESP32 a cada 2 segundos
    setInterval(function ( ) {
      var xhttp = new XMLHttpRequest();
      xhttp.onreadystatechange = function() {
        if (this.readyState == 4 && this.status == 200) {
          var data = JSON.parse(this.responseText);
          document.getElementById("v").innerHTML = data.v;
          document.getElementById("i").innerHTML = data.i;
          document.getElementById("p").innerHTML = data.p;
          document.getElementById("e").innerHTML = data.e;
        }
      };
      xhttp.open("GET", "/leituras", true);
      xhttp.send();
    }, 2000);
  </script>
</body>
</html>)rawliteral";

void setup() {
  Serial.begin(115200);

  // Inicia o Wi-Fi
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.println("Tentando conectar no Wi-Fi...");
  }
  
  // Mostra o IP no Monitor Serial (você vai digitar esse IP no navegador)
  Serial.println("");
  Serial.print("Conectado! Digite isso no navegador: ");
  Serial.println(WiFi.localIP());

  // Rota para abrir o site
  server.on("/", HTTP_GET, [](AsyncWebServerRequest *request){
    request->send_P(200, "text/html", index_html);
  });

  // Rota que envia os números pro site
  server.on("/leituras", HTTP_GET, [](AsyncWebServerRequest *request){
    float voltage = pzem.voltage();
    float current = pzem.current();
    float power = pzem.power();
    float energy = pzem.energy();

    // Se der erro na leitura (fiação solta), envia "0" para não bugar o site
    String json = "{";
    json += "\"v\":\"" + String(isnan(voltage) ? 0 : voltage, 1) + "\",";
    json += "\"i\":\"" + String(isnan(current) ? 0 : current, 2) + "\",";
    json += "\"p\":\"" + String(isnan(power) ? 0 : power, 1) + "\",";
    json += "\"e\":\"" + String(isnan(energy) ? 0 : energy, 3) + "\"";
    json += "}";
    
    request->send(200, "application/json", json);
  });

  server.begin();
}

void loop() {
  // Nada aqui, o servidor web cuida de tudo sozinho!
}
