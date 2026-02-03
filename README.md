# 🏋️ Sistema de Academia RFID com Google Sheets

Sistema completo de controle de acesso usando Arduino, RFID e Planilhas Google para registro de entrada e saída, esse sistema foi feito para 
uma pequena academia de bairro que precisava de um sistema assim, de controle.

## 🚀 Funcionalidades
- Cadastro de novos alunos via Web Serial API.
- Registro automático de Entrada (ON) e Saída (OFF) na planilha.
- Feedback em tempo real no Display LCD e Buzzer.
- Cálculo de estatísticas de presença.

## 📸 Imagem real do projeto
<p align="left">
  <img src="docs/Imagemreal.jpeg" width="50%">
</p>

## 🛠️ Hardware Utilizado
- Arduino Uno
- Leitor RFID RC522
- Display LCD 16x2 com I2C
- Buzzer Ativo (aqueles de pcs mesmo)
- Jumpers

## 💻 Tecnologias
- **Frontend:** HTML5, CSS3, JavaScript (Web Serial API).
- **Backend:** Google Apps Script (JavaScript).
- **Banco de Dados:** Google Sheets.
- **Firmware:** C++ (Arduino IDE).

### 🤖 Código HTML do projeto
```html
// Cole seu código HTML aqui
<!DOCTYPE html>
<!-- Define que é um documento HTML5 -->

<html lang="pt-br">
<!-- Idioma da página: português do Brasil -->

<head>
    <meta charset="UTF-8">
    <!-- Permite acentos, ç, etc -->

    <title>Sistema Academia - Cadastro RFID</title>
    <!-- Título que aparece na aba do navegador -->

    <style>
        /* ===== ESTILOS GERAIS DA PÁGINA ===== */

        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            padding: 30px;
            background-color: #f4f7f6;
        }

        /* Caixa principal centralizada */
        .container {
            max-width: 500px;
            background: white;
            padding: 20px;
            border-radius: 10px;
            box-shadow: 0 4px 6px rgba(0,0,0,0.1);
            margin: auto;
        }

        h2 {
            color: #333;
            text-align: center;
        }

        label {
            font-weight: bold;
            font-size: 0.9em;
            color: #555;
        }

        /* Campos de texto */
        input {
            margin-bottom: 15px;
            padding: 12px;
            width: calc(100% - 26px);
            border: 1px solid #ddd;
            border-radius: 5px;
            display: block;
            font-size: 1em;
        }

        /* Botões padrão */
        button {
            width: 100%;
            padding: 12px;
            cursor: pointer;
            border: none;
            border-radius: 5px;
            font-weight: bold;
            transition: 0.3s;
        }

        /* Botão conectar Arduino */
        .btn-connect {
            background-color: #007bff;
            color: white;
            margin-bottom: 20px;
        }

        /* Botão salvar */
        .btn-save {
            background-color: #28a745;
            color: white;
            font-size: 1.1em;
        }

        /* Texto de status da conexão */
        #status {
            text-align: center;
            font-size: 0.85em;
            margin-bottom: 20px;
            color: #666;
        }

        /* ===== MODAL (POPUP) ===== */

        .modal {
            display: none; /* começa escondido */
            position: fixed;
            z-index: 999;
            left: 0;
            top: 0;
            width: 100%;
            height: 100%;
            background: rgba(0,0,0,0.5);
            align-items: center;
            justify-content: center;
        }

        .modal-content {
            background: white;
            padding: 40px;
            border-radius: 15px;
            text-align: center;
            max-width: 400px;
            width: 80%;
            animation: popup 0.3s ease-out;
        }

        /* Animação de abrir popup */
        @keyframes popup {
            from { transform: scale(0.7); opacity: 0; }
            to { transform: scale(1); opacity: 1; }
        }

        .icon-sucesso { color: #28a745; font-size: 50px; }
        .icon-aviso { color: #ff9800; font-size: 50px; }
    </style>
</head>

<body>

<!-- ===== FORMULÁRIO PRINCIPAL ===== -->
<div class="container">

    <h2>Cadastro de Aluno</h2>

    <!-- Botão que conecta com o Arduino via USB -->
    <button class="btn-connect" onclick="conectarSerial()">
        1. Conectar Arduino (USB)
    </button>

    <!-- Mostra o status da conexão -->
    <div id="status">Status: Desconectado</div>

    <hr>

    <!-- Campo onde aparece o UID da tag -->
    <label>ID da Tag:</label>
    <input id="uid" placeholder="Aproxime a tag..." readonly
           style="background: #f9f9f9; border-color: #007bff;">

    <!-- Dados do aluno -->
    <input id="nome" placeholder="Nome Completo">
    <input id="cidade" placeholder="Cidade">
    <input id="bairro" placeholder="Bairro">
    <input id="peso" placeholder="Peso (ex: 80kg)">
    <input id="modalidade" placeholder="Modalidade">
    <input id="horario" placeholder="Horário">
    <input id="foto" placeholder="URL da foto">

    <!-- Botão salvar -->
    <button class="btn-save" onclick="enviar()">
        2. Salvar na Planilha
    </button>
</div>


<!-- ===== POPUP ===== -->
<div id="meuModal" class="modal">
    <div class="modal-content">
        <div id="modalIcon"></div>
        <h3 id="modalTitulo"></h3>
        <p id="modalTexto"></p>
    </div>
</div>


<script>
    // URL do Google Apps Script (planilha)
    const urlPlanilha = "URL DO SCRIPT";

    // Porta serial do Arduino
    let port;

    // Controla o loop de leitura
    let keepReading = true;



    // ===== CONECTA AO ARDUINO =====
    async function conectarSerial() {

        // Só funciona no Chrome/Edge
        if (!("serial" in navigator)) return alert("Use o Chrome!");

        try {
            // usuário escolhe a porta USB
            port = await navigator.serial.requestPort();

            // abre a comunicação em 9600 (igual Arduino)
            await port.open({ baudRate: 9600 });

            // atualiza status
            document.getElementById('status').innerText = "Status: Conectado! Aproxime a tag.";
            document.getElementById('status').style.color = "green";

            keepReading = true;

            // começa a ler os dados
            lerDados();

        } catch (e) {
            alert("Erro ao abrir porta: " + e);
        }
    }



    // ===== LÊ OS DADOS QUE VÊM DO ARDUINO =====
    async function lerDados() {

        // enquanto tiver porta ativa
        while (port.readable && keepReading) {

            const decoder = new TextDecoderStream();
            const readableStreamClosed = port.readable.pipeTo(decoder.writable);
            const reader = decoder.readable.getReader();

            let buffer = "";

            try {
                while (true) {
                    const { value, done } = await reader.read();
                    if (done) break;

                    buffer += value;

                    // quando Arduino manda \n = fim da leitura
                    if (buffer.includes("\n") || buffer.includes("\r")) {

                        let idLido = buffer.trim();

                        if (idLido.length >= 4) {
                            document.getElementById('uid').value = idLido;

                            // verifica se já existe na planilha
                            verificarExistencia(idLido);
                        }

                        buffer = "";
                    }
                }

            } finally {
                reader.releaseLock();
                await readableStreamClosed.catch(() => {});
            }

            await new Promise(resolve => setTimeout(resolve, 100));
        }
    }



    // ===== VERIFICA SE TAG JÁ EXISTE =====
    async function verificarExistencia(uid) {

        const res = await fetch(`${urlPlanilha}?uid=${uid}`);
        const data = await res.json();

        if (data.encontrado) {

            // mostra popup
            mostrarPopup("aviso", data.nome, "Status: " + data.status);

            document.getElementById('nome').value = data.nome;

            // envia info de volta ao Arduino
            if (port && port.writable) {
                const encoder = new TextEncoder();
                const writer = port.writable.getWriter();
                await writer.write(encoder.encode(data.nome + "|" + data.status + "\n"));
                writer.releaseLock();
            }

            setTimeout(() => {
                fecharPopup();
                limparTudo();
            }, 4000);
        }
    }



    // ===== ENVIA CADASTRO PARA PLANILHA =====
    function enviar() {

        let uid = document.getElementById('uid').value;
        let nome = document.getElementById('nome').value;

        if (!uid || !nome) return alert("Falta o ID ou o Nome!");

        let dados = {
            uid: uid,
            nome: nome,
            cidade: cidade.value,
            bairro: bairro.value,
            peso: peso.value,
            modalidade: modalidade.value,
            horario: horario.value,
            foto: foto.value
        };

        fetch(urlPlanilha, {
            method: "POST",
            mode: "no-cors",
            body: JSON.stringify(dados)
        });

        mostrarPopup("sucesso", "Sucesso!", "Cadastrado na planilha!");
    }



    // ===== POPUP =====
    function mostrarPopup(tipo, tit, txt) {

        const icon = document.getElementById('modalIcon');

        icon.innerHTML = tipo === "sucesso" ? "✅" : "⚠️";
        icon.className = tipo === "sucesso" ? "icon-sucesso" : "icon-aviso";

        modalTitulo.innerText = tit;
        modalTexto.innerText = txt;

        document.getElementById('meuModal').style.display = "flex";
    }

    function fecharPopup() {
        document.getElementById('meuModal').style.display = "none";
    }



    // ===== LIMPA TODOS OS CAMPOS =====
    function limparTudo() {
        document.querySelectorAll('input').forEach(i => i.value = "");
    }

</script>
</body>
</html>


### 🤖 Código do Arduino
```c++
#include <SPI.h>
#include <MFRC522.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

#define SS_PIN 10
#define RST_PIN 9
#define BUZZER_PIN 8 

MFRC522 rfid(SS_PIN, RST_PIN);
LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Serial.begin(9600);
  SPI.begin();
  rfid.PCD_Init();
  pinMode(BUZZER_PIN, OUTPUT); 

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0,0);
  lcd.print("Aproxime a tag");
}

void loop() {
  // VERIFICA SE O PC MANDOU DADOS (VINDO DO HTML)
  if (Serial.available() > 0) {
    String mensagem = Serial.readStringUntil('\n');
    mensagem.trim();

    lcd.clear();
    
    // VERIFICA SE A MENSAGEM É DE CADASTRO NOVO
    if (mensagem == "CADASTRO OK") {
      lcd.setCursor(0, 0);
      lcd.print("ALUNO SALVO!");
      lcd.setCursor(0, 1);
      lcd.print("COM SUCESSO!");

      // Som de sucesso (3 bips rápidos)
      for(int i=0; i<3; i++){
        tone(BUZZER_PIN, 3000); delay(70); noTone(BUZZER_PIN); delay(50);
      }
    } 
    // LÓGICA DE ENTRADA (ON) E SAÍDA (OFF)
    else {
      int divisor = mensagem.indexOf('|');
      if (divisor != -1) {
        String nome = mensagem.substring(0, divisor);
        String status = mensagem.substring(divisor + 1);

        lcd.setCursor(0, 0);
        lcd.print(nome.substring(0, 16));
        lcd.setCursor(0, 1);
        lcd.print("STATUS: ");
        lcd.print(status); 

        // Sons diferentes para Entrada e Saída
        if(status == "ON") {
          // Bip duplo agudo para ENTRADA
          tone(BUZZER_PIN, 2500); delay(100); noTone(BUZZER_PIN); delay(50);
          tone(BUZZER_PIN, 2500); delay(100); noTone(BUZZER_PIN);
        } else {
          // Bip longo mais grave para SAÍDA
          tone(BUZZER_PIN, 1500); delay(400); noTone(BUZZER_PIN);
        }
      } else {
        // Caso receba apenas o nome (fallback)
        lcd.setCursor(0, 0);
        lcd.print("BEM-VINDO:");
        lcd.setCursor(0, 1);
        lcd.print(mensagem.substring(0, 16));
      }
    }

    delay(3000); // Fica 3 segundos na tela
    lcd.clear();
    lcd.print("Aproxime a tag");
  }

  // LÓGICA NORMAL DE LEITURA DA TAG (RFID)
  if (!rfid.PICC_IsNewCardPresent()) return;
  if (!rfid.PICC_ReadCardSerial()) return;

  String uid = "";
  for (byte i = 0; i < rfid.uid.size; i++) {
    uid += String(rfid.uid.uidByte[i], HEX);
  }
  uid.toUpperCase();

  // Bip curto indicando que a tag foi lida
  tone(BUZZER_PIN, 2000); delay(100); noTone(BUZZER_PIN);

  Serial.println(uid); // Envia o ID para o HTML consultar

  lcd.clear();
  lcd.print("VERIFICANDO...");
  lcd.setCursor(0,1);
  lcd.print(uid);
  
  rfid.PICC_HaltA();
  rfid.PCD_StopCrypto1();
}
