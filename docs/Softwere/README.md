#include <WiFi.h>
#include <WebServer.h>
const char* ssid = "Carrinho-ESP32";
const char* password = "carrinho123";
WebServer server(80);

const int IN1 = 13;
const int IN2 = 12;
const int IN3 = 32;
const int IN4 = 33;
const int ENA = 14;
const int ENB = 27;

const int FREQUENCIA = 1000;
const int RESOLUCAO = 8;
const int VEL_80 = 204;
const int VEL_90 = 230;
const int VEL_100 = 255;
unsigned long ultimoComando = 0;
const unsigned long TEMPO_LIMITE = 700;

bool motoresLigados = false;
int velocidadeAtual = 0;

void parar() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);
  ledcWrite(ENA, 0);
  ledcWrite(ENB, 0);
  velocidadeAtual = 0;
  motoresLigados = false;
}

void definirVelocidade(int velocidade) {
  velocidade = constrain(velocidade, 0, 255);
  ledcWrite(ENA, velocidade);
  ledcWrite(ENB, velocidade);
  velocidadeAtual = velocidade;
}

void frente(int velocidade) {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
  definirVelocidade(velocidade);
  motoresLigados = true;
}

void tras(int velocidade) {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  definirVelocidade(velocidade);
  motoresLigados = true;
}

void esquerda(int velocidade) {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  definirVelocidade(velocidade);
  motoresLigados = true;
}

void direita(int velocidade) {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
  definirVelocidade(velocidade);
  motoresLigados = true;
}

void paginaInicial() {
  String html = R"rawliteral(
<!DOCTYPE html>
<html lang="pt-br">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <link rel="stylesheet" href="carrinho.css">
  <title>MCQUEEN'S</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      text-align: center;
      background-color: gray;
      margin: 0;
      padding: 50px;
      user-select: none;
    }
    h1 {
      color: #030303;
    }
    .painel {
      max-width: 360px;
      margin: auto;
      background-color: rgb(173, 173, 173);
      padding: 50px;
      border-radius: 1rem;
      box-shadow: 0 0 10px #ff0f0f, 0 0 20px #ff0f0f, 0 0 40px #ff0f0f;
    }
    .grade {
      display: grid;
      grid-template-columns: repeat(3, 1fr);
      gap: 10px;
      margin-top: 22px;
    }
    button {
      height: 82px;
      border: none;
      border-radius: 16px;
      font-family: monospace;
      font-size: 30px;
      font-weight: bold;
      color: rgb(175, 6, 6);
      background: #6c7281;
      touch-action: none;
    }
    button:active {
      background: #646a7c;
    }
    .parar {
      background: #e96666;
      font-size: 25px;
      font-family: monospace;
    }
    .vazio {
      visibility: hidden;
      font-family: monospace;
    }
    #estado {
      padding: 20px;
      padding-top: 20px;
      background-color: rgb(240, 233, 143);
      border-radius: 10px;
      margin-top: 40px;
      margin-left: 80px;
      margin-right: 80px;
      font-family: monospace;
      font-size: 20px;
    }
    .dica {
      font-size: 13px;
      color: #475569;
    }
  </style>
</head>

<body>
  <div class="painel">
    <h1>MCQUEEN'S</h1>
    <p>Segure a seta para acelerar!</p>

    <div class="grade">
      <div class="vazio"></div>
      <button data-m="F">⬆</button>
      <div class="vazio"></div>

      <button data-m="E">⬅</button>
      <button class="parar" data-m="S">PARAR</button>
      <button data-m="D">➞</button>

      <div class="vazio"></div>
      <button data-m="T">⬇</button>
      <div class="vazio"></div>
    </div>

    <div id="estado">Iniciar</div>
    <p class="dica"></p>
  </div>

<script>
let repeticao = null;
let movimentoAtual = "S";
let inicioMovimento = 0;
let versao = 0;
const estado = document.getElementById("estado");

async function enviar(comando, velocidade = 0) {
  try {
    const resposta = await fetch(
      "/cmd?m=" + comando + "&v=" + velocidade
    );
    if (!resposta.ok) throw new Error("Falha");
    if (comando === "S") {
      estado.textContent = "Carrinho parado";
    } else {
      estado.textContent =
        "Velocidade: " + Math.round(velocidade * 100 / 255) + "%";
    }
  } catch (e) {
    estado.textContent = "Falha de comunicacao";
  }
}

function pararAgora() {
  versao++;
  movimentoAtual = "S";

  if (repeticao !== null) {
    clearInterval(repeticao);
    repeticao = null;
  }
  enviar("S", 0);
}
function iniciar(comando) {
  if (repeticao !== null) {
    clearInterval(repeticao);
    repeticao = null;
  }
  versao++;
  const minhaVersao = versao;

  movimentoAtual = comando;
  inicioMovimento = Date.now();
  enviar(comando, 84);

  repeticao = setInterval(() => {
    if (versao !== minhaVersao ||
        movimentoAtual !== comando) return;
    const tempo = Date.now() - inicioMovimento;
    let velocidade;

    if (tempo < 1000) {
      velocidade = 204;   
    } else if (tempo < 2000) {
      velocidade = 230; 
    } else {
      velocidade = 255; 
    }

    enviar(comando, velocidade);
  }, 200);
}

document.querySelectorAll("button[data-m]").forEach(botao => {
  const comando = botao.dataset.m;

  botao.addEventListener("pointerdown", evento => {
    evento.preventDefault();

    if (comando === "S") {
      pararAgora();
    } else {
      iniciar(comando);
    }
  });

  botao.addEventListener("contextmenu",
    evento => evento.preventDefault());
});

window.addEventListener("pointerup", pararAgora);
window.addEventListener("pointercancel", pararAgora);
window.addEventListener("blur", pararAgora);

document.addEventListener("visibilitychange", () => {
  if (document.hidden) pararAgora();
});
</script>
</body>
</html>
)rawliteral";

  server.send(200, "text/html; charset=utf-8", html);
}

void receberComando() {
  if (!server.hasArg("m")) {
    parar();
    server.send(400, "text/plain", "Comando ausente");
    return;
  }
  String comando = server.arg("m");
  if (comando == "S") {
    parar();
    ultimoComando = millis();
    server.send(200, "text/plain", "OK");
    return;
  }

  if (comando != "F" && comando != "T" && comando != "E" && comando != "D") {
    parar();
    server.send(400, "text/plain", "Comando invalido");
    return;
  }

  int velocidade = 84;

  if (server.hasArg("v")) {
    velocidade = constrain(server.arg("v").toInt(), 84, 255);
  }
  if (comando == "F") {
    frente(velocidade);
  } else if (comando == "T") {
    tras(velocidade);
  } else if (comando == "E") {
    esquerda(velocidade);
  } else if (comando == "D") {
    direita(velocidade);
  }

  ultimoComando = millis();
  server.send(200, "text/plain", "OK");
}

void setup() {
  Serial.begin(115200);
  delay(500);

  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);

  bool pwmENA = ledcAttach(ENA, FREQUENCIA, RESOLUCAO);
  bool pwmENB = ledcAttach(ENB, FREQUENCIA, RESOLUCAO);
  parar();

  if (!pwmENA || !pwmENB) {
    Serial.println("Erro ao configurar PWM!");
    while (true) {
      parar();
      delay(1000);
    }
  }
  WiFi.mode(WIFI_AP);
  if (!WiFi.softAP(ssid, password)) {
    Serial.println("Erro ao criar a rede Wi-Fi!");
    parar();
    return;
  }
  server.on("/", HTTP_GET, paginaInicial);
  server.on("/cmd", HTTP_GET, receberComando);
  server.begin();
  Serial.println();
  Serial.println("REDE CRIADA!");
  Serial.print("Wi-Fi: ");
  Serial.println(ssid);
  Serial.print("Senha: ");
  Serial.println(password);
  Serial.print("Endereco: http://");
  Serial.println(WiFi.softAPIP());
  Serial.println("Servidor pronto!");
}
void loop() {
  server.handleClient();
  if (motoresLigados && millis() - ultimoComando > TEMPO_LIMITE) {
    parar();
  }
}