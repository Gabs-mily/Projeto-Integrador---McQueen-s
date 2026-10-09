#include <WiFi.h>
#include <WebServer.h>

// ===== REDE WIFI DO CARRINHO =====
const char* ssid = "Carrinho-ESP32";
const char* password = "carrinho123";

// ===== PINOS DO L298N =====
const int IN1 = 13;
const int IN2 = 12;
const int IN3 = 25;
const int IN4 = 26;

// ENA e ENB devem estar com os jumpers colocados.
WebServer server(80);

// Seguranca: para os motores se os comandos pararem.
unsigned long ultimoComando = 0;
const unsigned long TEMPO_LIMITE = 700;
bool motoresLigados = false;

// ===== CONTROLE DOS MOTORES =====
void parar() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);
  motoresLigados = false;
}

void frente() {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  motoresLigados = true;
}

void tras() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
  motoresLigados = true;
}

void esquerda() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  motoresLigados = true;
}

void direita() {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
  motoresLigados = true;
}

// ===== PAGINA DE CONTROLE =====
void paginaInicial() {
  String html = R"rawliteral(
<!DOCTYPE html>
<html lang="pt-br">
<head>
<meta charset="UTF-8">
<meta name="viewport"
      content="width=device-width, initial-scale=1,
      maximum-scale=1, user-scalable=no">
<title>Carrinho ESP32</title>
<style>
  body {
    font-family: Arial, sans-serif;
    text-align: center;
    background: #edf2f7;
    margin: 0;
    padding: 20px;
    user-select: none;
  }
  h1 { color: #172554; }
  .painel {
    max-width: 360px;
    margin: auto;
    background: white;
    padding: 18px;
    border-radius: 20px;
    box-shadow: 0 4px 15px #0002;
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
    font-size: 30px;
    font-weight: bold;
    color: white;
    background: #2563eb;
    touch-action: none;
  }
  button:active { background: #1d4ed8; }
  .parar {
    background: #dc2626;
    font-size: 17px;
  }
  .vazio { visibility: hidden; }
  #estado {
    padding: 12px;
    background: #e2e8f0;
    border-radius: 10px;
    margin-top: 18px;
  }
  .dica { font-size: 13px; color: #475569; }
</style>
</head>
<body>
<div class="painel">
  <h1>🚗 Meu Carrinho</h1>
  <p>Segure uma seta para movimentar.</p>

  <div class="grade">
    <div class="vazio"></div>
    <button data-m="F">⬆️</button>
    <div class="vazio"></div>

    <button data-m="E">⬅️</button>
    <button class="parar" data-m="S">PARAR</button>
    <button data-m="D">➡️</button>

    <div class="vazio"></div>
    <button data-m="T">⬇️</button>
    <div class="vazio"></div>
  </div>

  <div id="estado">Conectado à página</div>
  <p class="dica">
    Solte o botão para parar. Se a comunicação falhar,
    os motores param automaticamente.
  </p>
</div>

<script>
  let repeticao = null;
  let movimentoAtual = "S";
  let versao = 0;

  const estado = document.getElementById("estado");

  async function enviar(comando) {
    try {
      const resposta = await fetch("/cmd?m=" + comando);
      if (!resposta.ok) throw new Error("Falha");
      estado.textContent =
        comando === "S" ? "Carrinho parado" :
        "Comando enviado: " + comando;
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
    enviar("S");
  }

  function iniciar(comando) {
    pararAgora();
    const minhaVersao = versao;
    movimentoAtual = comando;
    enviar(comando);

    if (comando !== "S") {
      repeticao = setInterval(() => {
        if (versao === minhaVersao) enviar(comando);
      }, 200);
    }
  }

  document.querySelectorAll("button[data-m]").forEach(botao => {
    const comando = botao.dataset.m;

    botao.addEventListener("pointerdown", evento => {
      evento.preventDefault();
      if (comando === "S") pararAgora();
      else iniciar(comando);
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

// ===== RECEBER COMANDOS =====
void receberComando() {
  if (!server.hasArg("m")) {
    server.send(400, "text/plain", "Comando ausente");
    return;
  }

  String comando = server.arg("m");

  if (comando == "F") {
    frente();
  } else if (comando == "T") {
    tras();
  } else if (comando == "E") {
    esquerda();
  } else if (comando == "D") {
    direita();
  } else if (comando == "S") {
    parar();
  } else {
    parar();
    server.send(400, "text/plain", "Comando invalido");
    return;
  }

  ultimoComando = millis();
  server.send(200, "text/plain", "OK");
}

// ===== CONFIGURACAO =====
void setup() {
  Serial.begin(115200);
  delay(500);

  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);

  parar();

  WiFi.mode(WIFI_AP);

  if (!WiFi.softAP(ssid, password)) {
    Serial.println("Erro ao criar a rede Wi-Fi!");
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

  if (motoresLigados &&
      millis() - ultimoComando > TEMPO_LIMITE) {
    parar();
  }
}
