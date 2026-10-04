// =====================================================
//  TUXASSIST - ESP32-C3  (versão 2.1)
//  IA gratuita (Groq) com streaming, senha de acesso,
//  comandos especiais, limite de pedidos, voz e tema.
// =====================================================

#include <WiFi.h>
#include <WebServer.h>
#include <WiFiClientSecure.h>
#include <HTTPClient.h>
#include <Preferences.h>
#include <WiFiUdp.h>
#include <ESPmDNS.h>
#include <ctype.h>
#include <time.h>

// Validação de certificado HTTPS (item 5):
// instale a biblioteca "ESP32CertBundle" (tanakamasayuki) e ela ativa sozinha.
#if defined(__has_include)
  #if __has_include("esp32_cert_bundle.h")
    #include "esp32_cert_bundle.h"
    #define TEM_BUNDLE_CERT 1
  #endif
#endif

#ifdef TEM_BUNDLE_CERT
const bool VALIDANDO_CERTIFICADO = true;
#else
const bool VALIDANDO_CERTIFICADO = false;
#endif

// ---------------- Configurações ----------------
const int PORTA_DESCOBERTA = 4210;
const char* NOME_MDNS = "tuxassist";
const char* AP_SSID = "TUXASSIST-SETUP";
const char* AP_SENHA = "tuxassist";
const int LARGURA_TEXTO = 60;
const size_t MAX_PERGUNTA = 200;
const uint32_t TIMEOUT_HTTP_MS = 8000;
const uint32_t TIMEOUT_IA_MS = 15000;
const uint32_t INTERVALO_RECONEXAO_MS = 10000;
const char* USER_AGENT = "TUXASSIST/2.1 (ESP32-C3; projeto pessoal)";

// IA (Groq, API compatível com OpenAI).
// Os modelos são tentados em ordem: se o primeiro falhar, tenta o próximo.
// ATENÇÃO: llama-3.1-8b-instant e llama-3.3-70b-versatile foram desativados em 16/08/2026.
// Lista atual de modelos: console.groq.com/docs/models
const char* AI_URL = "https://api.groq.com/openai/v1/chat/completions";
const char* AI_MODELOS[] = { "openai/gpt-oss-20b", "openai/gpt-oss-120b" };
const size_t NUM_MODELOS = sizeof(AI_MODELOS) / sizeof(AI_MODELOS[0]);

// Fuso horário (Brasília = -3h)
const long GMT_OFFSET_S = -3 * 3600;

// Segurança e limites
const char* USUARIO_ACESSO = "admin";
const char* SENHA_INICIAL = "1234";
// true  = a cada boot a senha volta a ser SENHA_INICIAL
// false = usa a senha salva (a que você trocar em /config)
const bool RESETAR_SENHA_NO_BOOT = true;
const int MAX_FALHAS_LOGIN = 5;
const unsigned long BLOQUEIO_LOGIN_MS = 60000UL;
const int LIMITE_PEDIDOS_POR_MIN = 10;
const unsigned long INTERVALO_MIN_PEDIDOS_MS = 2000UL;

// ---------------- Globais ----------------
WebServer server(80);
Preferences preferencias;
WiFiUDP udp;

String ssid;
String senha;
String senhaAcesso;
String chaveAI;
String erroIA;
String linhaSerial;

bool modoConfig = false;
bool udpAtivo = false;
bool estavaConectado = false;
bool respostaIniciada = false;
unsigned long ultimaTentativa = 0;
int ultimoCodigoIA = 0;

int falhasLogin = 0;
unsigned long bloqueadoAte = 0;

struct Cliente {
  uint32_t ip;
  unsigned long inicioJanela;
  unsigned long ultimo;
  uint8_t cont;
};
Cliente clientes[4] = {};

// =====================================================
//  UTILIDADES DE TEXTO / JSON
// =====================================================

int valorHex(char c) {
  if (c >= '0' && c <= '9') return c - '0';
  if (c >= 'a' && c <= 'f') return c - 'a' + 10;
  if (c >= 'A' && c <= 'F') return c - 'A' + 10;
  return -1;
}

bool lerHex4(const String &s, int pos, uint32_t &valor) {
  if (pos + 4 > (int)s.length()) return false;
  uint32_t v = 0;
  for (int k = 0; k < 4; k++) {
    int h = valorHex(s[pos + k]);
    if (h < 0) return false;
    v = (v << 4) | (uint32_t)h;
  }
  valor = v;
  return true;
}

void anexarUtf8(String &saida, uint32_t cp) {
  if (cp < 0x80) {
    saida += (char)cp;
  } else if (cp < 0x800) {
    saida += (char)(0xC0 | (cp >> 6));
    saida += (char)(0x80 | (cp & 0x3F));
  } else if (cp < 0x10000) {
    saida += (char)(0xE0 | (cp >> 12));
    saida += (char)(0x80 | ((cp >> 6) & 0x3F));
    saida += (char)(0x80 | (cp & 0x3F));
  } else {
    saida += (char)(0xF0 | (cp >> 18));
    saida += (char)(0x80 | ((cp >> 12) & 0x3F));
    saida += (char)(0x80 | ((cp >> 6) & 0x3F));
    saida += (char)(0x80 | (cp & 0x3F));
  }
}

// Lê uma string JSON; "pos" aponta para o caractere DEPOIS da aspa de abertura.
String lerStringJson(const String &json, int pos) {
  String r;
  r.reserve(128);
  int n = json.length();
  int i = pos;

  while (i < n) {
    char c = json[i];

    if (c == '"') break;

    if (c == '\\' && i + 1 < n) {
      char e = json[i + 1];
      i += 2;

      switch (e) {
        case 'n': r += '\n'; break;
        case 'r': break;
        case 't': r += ' '; break;
        case 'u': {
          uint32_t cp;
          if (lerHex4(json, i, cp)) {
            i += 4;
            uint32_t baixo;
            if (cp >= 0xD800 && cp <= 0xDBFF &&
                i + 1 < n && json[i] == '\\' && json[i + 1] == 'u' &&
                lerHex4(json, i + 2, baixo) &&
                baixo >= 0xDC00 && baixo <= 0xDFFF) {
              cp = 0x10000 + ((cp - 0xD800) << 10) + (baixo - 0xDC00);
              i += 6;
            }
            anexarUtf8(r, cp);
          }
          break;
        }
        default:
          r += e;
          break;
      }
      continue;
    }

    r += c;
    i++;
  }

  return r;
}

int pularStringJson(const String &json, int pos) {
  int n = json.length();
  int i = pos;
  while (i < n) {
    char c = json[i];
    if (c == '\\') {
      i += 2;
      continue;
    }
    if (c == '"') return i + 1;
    i++;
  }
  return n;
}

String extrairJsonString(const String &json, const char *chave) {
  String marcador = String("\"") + chave + "\"";
  int n = json.length();
  int busca = 0;

  while (true) {
    int pos = json.indexOf(marcador, busca);
    if (pos < 0) return "";

    int i = pos + marcador.length();
    while (i < n && isspace((unsigned char)json[i])) i++;

    if (i < n && json[i] == ':') {
      i++;
      while (i < n && isspace((unsigned char)json[i])) i++;
      if (i < n && json[i] == '"') {
        return lerStringJson(json, i + 1);
      }
      return "";
    }

    busca = pos + marcador.length();
  }
}

// Busca "chave": 123.45 e devolve o número como texto (ignora valores que não são número).
String extrairJsonNumero(const String &json, const char *chave) {
  String marcador = String("\"") + chave + "\"";
  int n = json.length();
  int busca = 0;

  while (true) {
    int pos = json.indexOf(marcador, busca);
    if (pos < 0) return "";

    int i = pos + marcador.length();
    while (i < n && isspace((unsigned char)json[i])) i++;

    if (i < n && json[i] == ':') {
      i++;
      while (i < n && isspace((unsigned char)json[i])) i++;

      String num;
      while (i < n && (isdigit((unsigned char)json[i]) || json[i] == '-' || json[i] == '+' ||
                       json[i] == '.' || json[i] == 'e' || json[i] == 'E')) {
        num += json[i];
        i++;
      }
      if (num.length() > 0) return num;
    }

    busca = pos + marcador.length();
  }
}

String extrairPrimeiroResultadoWikipedia(const String &json) {
  int n = json.length();

  int i = json.indexOf('"');
  if (i < 0) return "";

  i = pularStringJson(json, i + 1);

  i = json.indexOf('[', i);
  if (i < 0) return "";
  i++;

  while (i < n && isspace((unsigned char)json[i])) i++;

  if (i >= n || json[i] != '"') return "";

  return lerStringJson(json, i + 1);
}

String jsonEscape(const String &s) {
  String r;
  r.reserve(s.length() + 16);
  for (int i = 0; i < (int)s.length(); i++) {
    char c = s[i];
    if (c == '"') r += "\\\"";
    else if (c == '\\') r += "\\\\";
    else if (c == '\n') r += "\\n";
    else if (c == '\r') continue;
    else if ((uint8_t)c < 0x20) r += ' ';
    else r += c;
  }
  return r;
}

String urlEncode(const String &texto) {
  String resultado;
  resultado.reserve(texto.length() * 3);
  const char *hex = "0123456789ABCDEF";

  for (int i = 0; i < (int)texto.length(); i++) {
    uint8_t c = (uint8_t)texto[i];

    if ((c >= 'a' && c <= 'z') ||
        (c >= 'A' && c <= 'Z') ||
        (c >= '0' && c <= '9') ||
        c == '-' || c == '_' || c == '.' || c == '~') {
      resultado += (char)c;
    } else {
      resultado += '%';
      resultado += hex[(c >> 4) & 0x0F];
      resultado += hex[c & 0x0F];
    }
  }

  return resultado;
}

// Minúsculas e sem acentos (para reconhecer comandos).
String normalizar(const String &s) {
  String r;
  r.reserve(s.length());
  int n = s.length();

  for (int i = 0; i < n; i++) {
    uint8_t c = (uint8_t)s[i];

    if (c == 0xC3 && i + 1 < n) {
      uint8_t d = (uint8_t)s[i + 1];
      i++;
      char o = '?';
      if ((d >= 0xA0 && d <= 0xA5) || (d >= 0x80 && d <= 0x85)) o = 'a';
      else if (d == 0xA7 || d == 0x87) o = 'c';
      else if ((d >= 0xA8 && d <= 0xAB) || (d >= 0x88 && d <= 0x8B)) o = 'e';
      else if ((d >= 0xAC && d <= 0xAF) || (d >= 0x8C && d <= 0x8F)) o = 'i';
      else if (d == 0xB1 || d == 0x91) o = 'n';
      else if ((d >= 0xB2 && d <= 0xB6) || (d >= 0x92 && d <= 0x96)) o = 'o';
      else if ((d >= 0xB9 && d <= 0xBC) || (d >= 0x99 && d <= 0x9C)) o = 'u';
      r += o;
      continue;
    }

    r += (char)tolower(c);
  }

  return r;
}

String virgula(String s) {
  s.replace('.', ',');
  return s;
}

String formatarResposta(const String &texto, int largura) {
  String resultado;
  resultado.reserve(texto.length() + texto.length() / largura + 8);
  int n = texto.length();
  int inicio = 0;

  while (inicio < n) {
    int fim = inicio + largura;

    if (fim >= n) {
      resultado += texto.substring(inicio);
      break;
    }

    int nl = texto.indexOf('\n', inicio);
    if (nl >= 0 && nl <= fim) {
      resultado += texto.substring(inicio, nl + 1);
      inicio = nl + 1;
      continue;
    }

    int espaco = texto.lastIndexOf(' ', fim);

    if (espaco <= inicio) {
      while (fim > inicio + 1 && (((uint8_t)texto[fim]) & 0xC0) == 0x80) {
        fim--;
      }
      resultado += texto.substring(inicio, fim);
      resultado += '\n';
      inicio = fim;
    } else {
      resultado += texto.substring(inicio, espaco);
      resultado += '\n';
      inicio = espaco + 1;
    }

    while (inicio < n && texto[inicio] == ' ') {
      inicio++;
    }
  }

  return resultado;
}

// =====================================================
//  FRASES / PERGUNTAS (Wikipedia)
// =====================================================

bool ehFimDeFrase(const String &t, int i) {
  char c = t[i];
  if (c != '.' && c != '!' && c != '?') return false;

  int n = t.length();
  if (i + 1 >= n) return true;
  if (t[i + 1] != ' ') return false;

  if (c == '.' && i >= 1 && isalpha((unsigned char)t[i - 1]) &&
      (i == 1 || t[i - 2] == ' ' || t[i - 2] == '.')) {
    return false;
  }

  int j = i + 1;
  while (j < n && t[j] == ' ') j++;
  if (j >= n) return true;

  uint8_t p = (uint8_t)t[j];
  return isupper(p) || isdigit(p) || p >= 0x80 || p == '"' || p == '(';
}

String cortarFrases(const String &texto, int frases) {
  int contagem = 0;
  for (int i = 0; i < (int)texto.length(); i++) {
    if (ehFimDeFrase(texto, i)) {
      contagem++;
      if (contagem >= frases) {
        return texto.substring(0, i + 1);
      }
    }
  }
  return texto;
}

String extrairFraseCriacao(const String &texto) {
  String minusculo = texto;
  minusculo.toLowerCase();

  const char *chaves[] = {
    "criado por", "criada por", "foi criado", "foi criada",
    "desenvolvido por", "desenvolvida por",
    "fundado por", "fundada por",
    "inventado por", "inventada por"
  };

  int pos = -1;
  for (size_t k = 0; k < sizeof(chaves) / sizeof(chaves[0]); k++) {
    pos = minusculo.indexOf(chaves[k]);
    if (pos >= 0) break;
  }

  if (pos < 0) return "";

  int n = texto.length();

  int inicio = 0;
  for (int i = pos; i > 0; i--) {
    if (ehFimDeFrase(texto, i - 1)) {
      inicio = i;
      break;
    }
  }
  while (inicio < n && texto[inicio] == ' ') inicio++;

  int fim = n;
  for (int i = pos; i < n; i++) {
    if (ehFimDeFrase(texto, i)) {
      fim = i + 1;
      break;
    }
  }

  String frase = texto.substring(inicio, fim);
  frase.trim();
  return frase;
}

bool perguntaSobreCriador(const String &pergunta) {
  String p = pergunta;
  p.toLowerCase();

  bool temCriador =
    p.indexOf("criador") >= 0 ||
    p.indexOf("criou") >= 0 ||
    p.indexOf("criado") >= 0 ||
    p.indexOf("inventor") >= 0 ||
    p.indexOf("inventou") >= 0 ||
    p.indexOf("fundador") >= 0 ||
    p.indexOf("fundou") >= 0;

  return temCriador && p.indexOf("quem") >= 0;
}

String limparPergunta(const String &original) {
  String t = original;
  t.trim();

  String minus = t;
  minus.toLowerCase();

  const char *prefixos[] = {
    "quem foi o criador do ", "quem foi o criador da ", "quem foi o criador de ",
    "quem foi a criadora do ", "quem foi a criadora da ",
    "quem é o criador do ", "quem é o criador da ", "quem é o criador de ",
    "quem e o criador do ", "quem e o criador da ", "quem e o criador de ",
    "quem criou o ", "quem criou a ", "quem criou ",
    "quem inventou o ", "quem inventou a ", "quem inventou ",
    "criador do ", "criador da ", "criador de ",
    "quem foi ", "quem é ", "quem e ",
    "o que é ", "o que e ", "o que foi ", "o que são ", "o que sao ",
    "o que significa ", "me fale sobre ", "fale sobre ",
    "me explique ", "explique "
  };

  bool removeuPrefixo = false;
  for (size_t k = 0; k < sizeof(prefixos) / sizeof(prefixos[0]); k++) {
    String p = prefixos[k];
    if (minus.startsWith(p)) {
      t = t.substring(p.length());
      removeuPrefixo = true;
      break;
    }
  }

  if (removeuPrefixo) {
    String m = t;
    m.toLowerCase();
    const char *artigos[] = { "o ", "a ", "os ", "as ", "um ", "uma " };
    for (size_t k = 0; k < sizeof(artigos) / sizeof(artigos[0]); k++) {
      String a = artigos[k];
      if (m.startsWith(a)) {
        t = t.substring(a.length());
        break;
      }
    }
  }

  t.trim();
  while (t.length() > 0 &&
         (t.endsWith("?") || t.endsWith(".") || t.endsWith("!") || t.endsWith(" "))) {
    t.remove(t.length() - 1);
  }

  if (t.length() == 0) return original;

  if (t[0] >= 'a' && t[0] <= 'z') {
    t.setCharAt(0, (char)toupper(t[0]));
  }

  return t;
}

// =====================================================
//  HORA (NTP)
// =====================================================

bool horaValida() {
  return time(nullptr) > 1700000000;
}

bool obterHora(struct tm &info) {
  if (!horaValida()) return false;
  return getLocalTime(&info, 200);
}

// =====================================================
//  HTTPS / TLS
// =====================================================

void configurarTLS(WiFiClientSecure &c) {
#ifdef TEM_BUNDLE_CERT
  #if defined(ESP_ARDUINO_VERSION_MAJOR) && ESP_ARDUINO_VERSION_MAJOR >= 3
    c.setCACertBundle(x509_crt_bundle, x509_crt_bundle_len);
  #else
    c.setCACertBundle(x509_crt_bundle);
  #endif
#else
  c.setInsecure();
#endif
}

bool httpGetString(const String &url, String &corpo) {
  if (WiFi.status() != WL_CONNECTED) return false;

  if (VALIDANDO_CERTIFICADO && !horaValida()) {
    Serial.println("Relogio ainda nao sincronizado (necessario para validar HTTPS).");
    return false;
  }

  WiFiClientSecure cliente;
  configurarTLS(cliente);

  HTTPClient http;
  http.setConnectTimeout(TIMEOUT_HTTP_MS);
  http.setTimeout(TIMEOUT_HTTP_MS);
  http.setUserAgent(USER_AGENT);
  http.setFollowRedirects(HTTPC_STRICT_FOLLOW_REDIRECTS);

  if (!http.begin(cliente, url)) return false;

  http.addHeader("Accept", "application/json");

  int codigo = http.GET();

  if (codigo != HTTP_CODE_OK) {
    Serial.print("HTTP erro/codigo: ");
    Serial.println(codigo);
    http.end();
    return false;
  }

  corpo = http.getString();
  http.end();

  return corpo.length() > 0;
}

// =====================================================
//  COMANDOS ESPECIAIS (item 14)
// =====================================================

struct Moeda {
  const char *palavra;
  const char *codigo;
};

const Moeda MOEDAS[] = {
  { "dolar", "USD" }, { "dolares", "USD" }, { "usd", "USD" },
  { "euro", "EUR" }, { "euros", "EUR" }, { "eur", "EUR" },
  { "real", "BRL" }, { "reais", "BRL" }, { "brl", "BRL" },
  { "libra", "GBP" }, { "libras", "GBP" }, { "gbp", "GBP" },
  { "iene", "JPY" }, { "ienes", "JPY" }, { "jpy", "JPY" }
};
const size_t NUM_MOEDAS = sizeof(MOEDAS) / sizeof(MOEDAS[0]);

String descricaoClima(int c) {
  if (c == 0) return "céu limpo";
  if (c == 1) return "predominantemente limpo";
  if (c == 2) return "parcialmente nublado";
  if (c == 3) return "nublado";
  if (c == 45 || c == 48) return "neblina";
  if (c >= 51 && c <= 57) return "garoa";
  if (c >= 61 && c <= 67) return "chuva";
  if (c >= 71 && c <= 77) return "neve";
  if (c >= 80 && c <= 82) return "pancadas de chuva";
  if (c == 85 || c == 86) return "pancadas de neve";
  if (c == 95) return "trovoada";
  if (c == 96 || c == 99) return "trovoada com granizo";
  return "condição desconhecida";
}

void comandoClima(String cidade, String &saida) {
  cidade.trim();

  const char *sufixos[] = { " hoje", " agora" };
  for (size_t k = 0; k < 2; k++) {
    if (cidade.endsWith(sufixos[k])) {
      cidade.remove(cidade.length() - strlen(sufixos[k]));
    }
  }
  cidade.trim();

  if (cidade.length() == 0) {
    saida = "Diga a cidade. Exemplo: clima em Curitiba";
    return;
  }

  String dados;

  String urlGeo =
    "https://geocoding-api.open-meteo.com/v1/search?count=1&language=pt&format=json&name=" +
    urlEncode(cidade);

  if (!httpGetString(urlGeo, dados)) {
    saida = "Não consegui consultar o serviço de clima agora.";
    return;
  }

  String lat = extrairJsonNumero(dados, "latitude");
  String lon = extrairJsonNumero(dados, "longitude");

  if (lat.length() == 0 || lon.length() == 0) {
    saida = "Não encontrei a cidade \"" + cidade + "\".";
    return;
  }

  String nome = extrairJsonString(dados, "name");
  String regiao = extrairJsonString(dados, "admin1");

  String urlClima =
    "https://api.open-meteo.com/v1/forecast?latitude=" + lat +
    "&longitude=" + lon +
    "&current=temperature_2m,apparent_temperature,relative_humidity_2m,wind_speed_10m,weather_code"
    "&timezone=auto";

  if (!httpGetString(urlClima, dados)) {
    saida = "Não consegui obter a previsão agora.";
    return;
  }

  String temp = extrairJsonNumero(dados, "temperature_2m");
  String sens = extrairJsonNumero(dados, "apparent_temperature");
  String umid = extrairJsonNumero(dados, "relative_humidity_2m");
  String vento = extrairJsonNumero(dados, "wind_speed_10m");
  String codigo = extrairJsonNumero(dados, "weather_code");

  if (temp.length() == 0) {
    saida = "Não consegui ler a previsão desta vez.";
    return;
  }

  saida = nome;
  if (regiao.length() > 0) {
    saida += ", ";
    saida += regiao;
  }
  saida += ": ";
  saida += virgula(temp);
  saida += " °C";
  if (sens.length() > 0) {
    saida += " (sensação de ";
    saida += virgula(sens);
    saida += " °C)";
  }
  saida += ", ";
  saida += descricaoClima(codigo.toInt());
  saida += ".";
  if (umid.length() > 0) {
    saida += " Umidade ";
    saida += umid;
    saida += "%.";
  }
  if (vento.length() > 0) {
    saida += " Vento ";
    saida += virgula(vento);
    saida += " km/h.";
  }
}

// n = texto normalizado. Retorna true se tratou como comando de moeda.
bool comandoMoeda(const String &n, String &saida) {
  int posA = -1, posB = -1;
  const char *codA = nullptr;
  const char *codB = nullptr;

  for (size_t k = 0; k < NUM_MOEDAS; k++) {
    int p = n.indexOf(MOEDAS[k].palavra);
    if (p >= 0 && (posA < 0 || p < posA)) {
      posA = p;
      codA = MOEDAS[k].codigo;
    }
  }

  if (codA != nullptr) {
    for (size_t k = 0; k < NUM_MOEDAS; k++) {
      if (strcmp(MOEDAS[k].codigo, codA) == 0) continue;
      int p = n.indexOf(MOEDAS[k].palavra);
      if (p >= 0 && (posB < 0 || p < posB)) {
        posB = p;
        codB = MOEDAS[k].codigo;
      }
    }
  }

  if (codA == nullptr) {
    saida = "Qual moeda? Exemplo: cotação do dólar, ou converter 100 dólar para real";
    return true;
  }

  if (codB == nullptr) {
    codB = (strcmp(codA, "BRL") == 0) ? "USD" : "BRL";
  }

  float valor = 1.0f;
  for (int i = 0; i < (int)n.length(); i++) {
    if (isdigit((unsigned char)n[i])) {
      String num;
      int j = i;
      while (j < (int)n.length() && (isdigit((unsigned char)n[j]) || n[j] == ',' || n[j] == '.')) {
        num += (n[j] == ',') ? '.' : n[j];
        j++;
      }
      valor = num.toFloat();
      break;
    }
  }

  String dados;
  String url = String("https://open.er-api.com/v6/latest/") + codA;

  if (!httpGetString(url, dados)) {
    saida = "Não consegui consultar a cotação agora.";
    return true;
  }

  float taxa = extrairJsonNumero(dados, codB).toFloat();

  if (taxa <= 0) {
    saida = "Não consegui obter essa cotação.";
    return true;
  }

  saida = virgula(String(valor, 2));
  saida += " ";
  saida += codA;
  saida += " = ";
  saida += virgula(String(valor * taxa, 2));
  saida += " ";
  saida += codB;
  saida += " (taxa aproximada)";
  return true;
}

// Retorna true se a pergunta era um comando e "saida" foi preenchida.
bool tratarComando(const String &pergunta, String &saida) {
  String n = normalizar(pergunta);
  n.trim();
  while (n.length() > 0 && (n.endsWith("?") || n.endsWith("!") || n.endsWith("."))) {
    n.remove(n.length() - 1);
  }
  n.trim();

  if (n == "ajuda" || n == "comandos" || n == "help") {
    saida =
      "Comandos:\n"
      "- que horas são\n"
      "- que dia é hoje\n"
      "- clima em <cidade>\n"
      "- cotação do dólar\n"
      "- converter 100 dólar para real\n"
      "Qualquer outra pergunta vai para a IA.";
    return true;
  }

  if (n == "hora" || n == "horas" || n.indexOf("que horas") >= 0) {
    struct tm info;
    if (!obterHora(info)) {
      saida = "O relógio ainda está sincronizando. Tente de novo em instantes.";
      return true;
    }
    char buf[40];
    snprintf(buf, sizeof(buf), "Agora são %02d:%02d.", info.tm_hour, info.tm_min);
    saida = buf;
    return true;
  }

  if (n == "data" || n.indexOf("que dia e hoje") >= 0 ||
      n.indexOf("data de hoje") >= 0 || n.indexOf("dia de hoje") >= 0) {
    struct tm info;
    if (!obterHora(info)) {
      saida = "O relógio ainda está sincronizando. Tente de novo em instantes.";
      return true;
    }
    const char *dias[] = { "domingo", "segunda-feira", "terça-feira", "quarta-feira",
                           "quinta-feira", "sexta-feira", "sábado" };
    char buf[64];
    snprintf(buf, sizeof(buf), "Hoje é %s, %02d/%02d/%04d.",
             dias[info.tm_wday], info.tm_mday, info.tm_mon + 1, info.tm_year + 1900);
    saida = buf;
    return true;
  }

  bool falaDeClima =
    n.indexOf("clima") >= 0 || n.indexOf("previsao") >= 0 ||
    n.startsWith("tempo em ") || n.indexOf("temperatura") >= 0;

  if (falaDeClima) {
    int p = n.indexOf(" em ");
    int tamanho = 4;

    if (p < 0 && (n.indexOf("clima") >= 0 || n.indexOf("previsao") >= 0)) {
      p = n.indexOf(" de ");
    }
    if (n.startsWith("tempo em ")) {
      p = 5;
    }

    if (p >= 0) {
      comandoClima(n.substring(p + tamanho), saida);
      return true;
    }

    if (n.indexOf("clima") >= 0 || n.indexOf("previsao") >= 0) {
      saida = "Diga a cidade. Exemplo: clima em Curitiba";
      return true;
    }
  }

  bool falaDeMoeda =
    n.indexOf("cotacao") >= 0 || n.startsWith("converter") || n.startsWith("converta");

  if (falaDeMoeda) {
    return comandoMoeda(n, saida);
  }

  if (n.indexOf("quanto vale") >= 0 || n.indexOf("quanto custa") >= 0) {
    for (size_t k = 0; k < NUM_MOEDAS; k++) {
      if (n.indexOf(MOEDAS[k].palavra) >= 0) {
        return comandoMoeda(n, saida);
      }
    }
  }

  return false;
}

// =====================================================
//  PESQUISAS (Wikipedia / DuckDuckGo)
// =====================================================

String pegarResumoWikipedia(const String &pergunta, int frases) {
  String busca = limparPergunta(pergunta);

  Serial.println("Pesquisando Wikipedia...");
  Serial.print("Busca: ");
  Serial.println(busca);

  String url =
    "https://pt.wikipedia.org/w/api.php"
    "?action=opensearch"
    "&search=" + urlEncode(busca) +
    "&limit=1"
    "&namespace=0"
    "&format=json"
    "&utf8=1";

  String dados;

  if (!httpGetString(url, dados)) return "";

  String titulo = extrairPrimeiroResultadoWikipedia(dados);

  if (titulo.length() == 0) return "";

  Serial.print("Artigo encontrado: ");
  Serial.println(titulo);

  titulo.replace(" ", "_");

  String resumoURL =
    "https://pt.wikipedia.org/api/rest_v1/page/summary/" + urlEncode(titulo);

  if (!httpGetString(resumoURL, dados)) return "";

  if (dados.indexOf("\"type\":\"disambiguation\"") >= 0) return "";

  String texto = extrairJsonString(dados, "extract");

  if (texto.length() == 0) return "";

  if (perguntaSobreCriador(pergunta)) {
    String frase = extrairFraseCriacao(texto);
    if (frase.length() > 0) return frase;
  }

  return cortarFrases(texto, frases);
}

String pesquisarDuckDuckGo(const String &pergunta) {
  Serial.println("Tentando DuckDuckGo...");

  String url =
    "https://api.duckduckgo.com/?q=" + urlEncode(limparPergunta(pergunta)) +
    "&format=json"
    "&no_html=1"
    "&no_redirect=1"
    "&skip_disambig=1";

  String dados;

  if (!httpGetString(url, dados)) return "";

  const char *campos[] = { "Answer", "AbstractText", "Definition" };

  for (size_t k = 0; k < sizeof(campos) / sizeof(campos[0]); k++) {
    String resposta = extrairJsonString(dados, campos[k]);
    if (resposta.length() > 0) return resposta;
  }

  return "";
}

// =====================================================
//  RESPOSTA EM STREAMING (item 2)
// =====================================================

void iniciarResposta() {
  if (respostaIniciada) return;
  respostaIniciada = true;
  server.setContentLength(CONTENT_LENGTH_UNKNOWN);
  server.sendHeader("Cache-Control", "no-cache");
  server.send(200, "text/plain; charset=utf-8", "");
}

void enviarTexto(const String &t) {
  if (t.length() == 0) return;
  iniciarResposta();
  server.sendContent(t);
}

void finalizarResposta() {
  iniciarResposta();
  server.sendContent("");
  respostaIniciada = false;
}

void enviarCompleto(const String &t) {
  enviarTexto(t);
  finalizarResposta();
}

// =====================================================
//  IA (item 1) - Groq, com streaming SSE
// =====================================================

String promptSistema(bool longa) {
  String p =
    "Você é o TUXASSIST, um assistente que roda em um ESP32-C3. "
    "Responda sempre em português do Brasil, em texto simples, sem markdown, "
    "sem asteriscos e sem listas com símbolos. Seja correto e direto. "
    "Se não souber, diga que não sabe. ";
  p += longa ? "Explique com mais detalhes, em até 3 parágrafos curtos."
             : "Seja breve: no máximo 3 frases.";
  return p;
}

// Consulta UM modelo da Groq. Retorna true se enviou alguma resposta ao navegador.
// Em falha, preenche erroIA e ultimoCodigoIA.
bool consultarGroq(const char *modelo, const String &pergunta, bool longa) {
  ultimoCodigoIA = 0;

  Serial.print("Consultando IA (");
  Serial.print(modelo);
  Serial.println(")...");

  WiFiClientSecure cliente;
  configurarTLS(cliente);

  HTTPClient http;
  http.useHTTP10(true);   // sem "chunked": facilita ler o stream
  http.setConnectTimeout(TIMEOUT_HTTP_MS);
  http.setTimeout(TIMEOUT_IA_MS);
  http.setUserAgent(USER_AGENT);

  if (!http.begin(cliente, AI_URL)) {
    erroIA = "falha ao iniciar a conexão";
    return false;
  }

  http.addHeader("Content-Type", "application/json");
  http.addHeader("Authorization", String("Bearer ") + chaveAI);

  String corpo = "{\"model\":\"";
  corpo += modelo;
  corpo += "\",\"stream\":true,\"temperature\":0.4,\"reasoning_effort\":\"low\",\"max_completion_tokens\":";
  corpo += (longa ? 1200 : 600);
  corpo += ",\"messages\":[{\"role\":\"system\",\"content\":\"";
  corpo += jsonEscape(promptSistema(longa));
  corpo += "\"},{\"role\":\"user\",\"content\":\"";
  corpo += jsonEscape(pergunta);
  corpo += "\"}]}";

  int codigo = http.POST(corpo);
  ultimoCodigoIA = codigo;

  if (codigo != HTTP_CODE_OK) {
    Serial.print("IA codigo: ");
    Serial.println(codigo);

    String msg = "";
    if (codigo > 0) {
      msg = extrairJsonString(http.getString(), "message");
      if (msg.length() > 120) msg = msg.substring(0, 120);
    }

    if (codigo == 401 || codigo == 403) erroIA = "chave da API inválida";
    else if (codigo == 429) erroIA = "limite gratuito da IA atingido, tente de novo em instantes";
    else if (codigo <= 0) erroIA = "falha de conexão com a IA (confira a hora e o certificado)";
    else if (msg.length() > 0) erroIA = msg;
    else erroIA = "erro " + String(codigo);

    http.end();
    return false;
  }

  auto *stream = http.getStreamPtr();

  unsigned long inicio = millis();
  unsigned long ultimoDado = millis();
  bool enviou = false;
  bool concluiu = false;

  while (millis() - inicio < 60000UL) {
    if (stream->available()) {
      String linha = stream->readStringUntil('\n');
      ultimoDado = millis();
      linha.trim();

      if (!linha.startsWith("data:")) continue;

      String payload = linha.substring(5);
      payload.trim();

      if (payload == "[DONE]") {
        concluiu = true;
        break;
      }

      String pedaco = extrairJsonString(payload, "content");

      if (pedaco.length() > 0) {
        enviarTexto(pedaco);
        enviou = true;
      }
    } else {
      if (!stream->connected()) {
        concluiu = true;
        break;
      }
      if (millis() - ultimoDado > 20000UL) break;
      delay(5);
    }
  }

  http.end();

  if (enviou && !concluiu) {
    enviarTexto("\n[resposta interrompida]");
  }

  if (!enviou) {
    erroIA = "a IA não devolveu texto";
  }

  return enviou;
}

// Tenta cada modelo da lista, em ordem. Retorna true se algum respondeu.
bool perguntarIA(const String &pergunta, bool longa) {
  if (chaveAI.length() == 0) return false;

  if (WiFi.status() != WL_CONNECTED) {
    erroIA = "sem internet";
    return false;
  }

  if (VALIDANDO_CERTIFICADO && !horaValida()) {
    erroIA = "relógio ainda não sincronizado";
    return false;
  }

  for (size_t k = 0; k < NUM_MODELOS; k++) {
    if (consultarGroq(AI_MODELOS[k], pergunta, longa)) return true;

    // chave errada: não adianta tentar outro modelo
    if (ultimoCodigoIA == 401 || ultimoCodigoIA == 403) break;
  }

  return false;
}

// =====================================================
//  WI-FI
// =====================================================

void conectarWiFi() {
  Serial.println();
  Serial.println("================================");
  Serial.println("       CONECTANDO AO WI-FI");
  Serial.println("================================");

  Serial.print("Rede: ");
  Serial.println(ssid);

  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid.c_str(), senha.c_str());

  int tentativas = 0;

  while (WiFi.status() != WL_CONNECTED && tentativas < 30) {
    delay(500);
    Serial.print(".");
    tentativas++;
  }

  Serial.println();

  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("Wi-Fi conectado!");
    Serial.print("IP do ESP32: ");
    Serial.println(WiFi.localIP());
    Serial.print("RSSI: ");
    Serial.print(WiFi.RSSI());
    Serial.println(" dBm");
  } else {
    Serial.println("Nao foi possivel conectar ao Wi-Fi.");
  }
}

void iniciarModoConfig() {
  modoConfig = true;

  WiFi.mode(WIFI_AP_STA);
  WiFi.softAP(AP_SSID, AP_SENHA);

  Serial.println();
  Serial.println("MODO CONFIGURACAO ATIVO");
  Serial.print("Conecte-se na rede: ");
  Serial.println(AP_SSID);
  Serial.print("Abra: http://");
  Serial.print(WiFi.softAPIP());
  Serial.println("/wifi");

  if (ssid.length() > 0) {
    WiFi.begin(ssid.c_str(), senha.c_str());
  }

  ultimaTentativa = millis();
}

void iniciarDescoberta() {
  udp.stop();
  udpAtivo = udp.begin(PORTA_DESCOBERTA) == 1;

  if (udpAtivo) {
    Serial.print("Descoberta UDP iniciada na porta ");
    Serial.println(PORTA_DESCOBERTA);
  } else {
    Serial.println("Falha ao iniciar descoberta UDP.");
  }
}

void aoConectar() {
  Serial.print("IP: http://");
  Serial.println(WiFi.localIP());

  configTime(GMT_OFFSET_S, 0, "pool.ntp.org", "time.google.com");

  MDNS.end();
  if (MDNS.begin(NOME_MDNS)) {
    MDNS.addService("http", "tcp", 80);
    Serial.print("mDNS: http://");
    Serial.print(NOME_MDNS);
    Serial.println(".local");
  }

  iniciarDescoberta();

  if (modoConfig) {
    modoConfig = false;
    WiFi.softAPdisconnect(true);
    Serial.println("Modo configuracao encerrado.");
  }
}

void verificarWiFi() {
  bool conectado = WiFi.status() == WL_CONNECTED;

  if (conectado && !estavaConectado) {
    estavaConectado = true;
    Serial.println("Wi-Fi conectado/reconectado!");
    aoConectar();
  } else if (!conectado && estavaConectado) {
    estavaConectado = false;
    udp.stop();
    udpAtivo = false;
    ultimaTentativa = millis();
    Serial.println("Wi-Fi desconectado.");
  }

  if (!conectado && ssid.length() > 0 &&
      millis() - ultimaTentativa >= INTERVALO_RECONEXAO_MS) {
    ultimaTentativa = millis();
    Serial.println("Tentando reconectar...");
    WiFi.disconnect();
    WiFi.begin(ssid.c_str(), senha.c_str());
  }
}

// =====================================================
//  DESCOBERTA UDP
// =====================================================

void verificarDescoberta() {
  if (!udpAtivo) return;

  int tamanho = udp.parsePacket();
  if (tamanho <= 0) return;

  char pacote[100];
  int quantidade = udp.read(pacote, sizeof(pacote) - 1);
  if (quantidade <= 0) return;

  pacote[quantidade] = '\0';

  String mensagem = String(pacote);
  mensagem.trim();

  if (mensagem == "TUXASSIST_DISCOVER") {
    String resposta;
    resposta += "TUXASSIST\n";
    resposta += "IP=";
    resposta += WiFi.localIP().toString();
    resposta += "\nPORT=80\n";

    udp.beginPacket(udp.remoteIP(), udp.remotePort());
    udp.print(resposta);
    udp.endPacket();
  }
}

// =====================================================
//  SENHA DE ACESSO + LIMITE DE PEDIDOS (itens 6 e 18)
// =====================================================

String gerarSenha(int tamanho) {
  const char alfabeto[] = "abcdefghjkmnpqrstuvwxyz23456789";
  String s;
  for (int i = 0; i < tamanho; i++) {
    s += alfabeto[esp_random() % (sizeof(alfabeto) - 1)];
  }
  return s;
}

void mostrarSenhaNoSerial() {
  Serial.println();
  Serial.println("================================");
  Serial.print("USUARIO: ");
  Serial.println(USUARIO_ACESSO);
  Serial.print("SENHA DE ACESSO: ");
  Serial.println(senhaAcesso);
  Serial.println("================================");
}

bool autorizado() {
  unsigned long agora = millis();

  if (bloqueadoAte != 0 && (long)(bloqueadoAte - agora) > 0) {
    server.sendHeader("Retry-After", String((bloqueadoAte - agora) / 1000 + 1));
    server.send(429, "text/plain; charset=utf-8", "Muitas tentativas de login. Aguarde um minuto.");
    return false;
  }

  if (server.authenticate(USUARIO_ACESSO, senhaAcesso.c_str())) {
    falhasLogin = 0;
    return true;
  }

  if (server.hasHeader("Authorization")) {
    falhasLogin++;
    if (falhasLogin >= MAX_FALHAS_LOGIN) {
      falhasLogin = 0;
      bloqueadoAte = agora + BLOQUEIO_LOGIN_MS;
      Serial.println("Muitas tentativas de login: acesso bloqueado por 1 minuto.");
    }
  }

  server.requestAuthentication(BASIC_AUTH, "TUXASSIST", "Acesso negado.");
  return false;
}

bool dentroDoLimite(IPAddress ip, unsigned long &esperaMs) {
  unsigned long agora = millis();
  uint32_t chave = (uint32_t)ip;

  int idx = -1;
  int maisAntigo = 0;

  for (int i = 0; i < 4; i++) {
    if (clientes[i].ip == chave) {
      idx = i;
      break;
    }
    if (clientes[i].ultimo < clientes[maisAntigo].ultimo) maisAntigo = i;
  }

  if (idx < 0) {
    idx = maisAntigo;
    clientes[idx] = { chave, agora, agora - INTERVALO_MIN_PEDIDOS_MS, 0 };
  }

  Cliente &c = clientes[idx];

  if (agora - c.inicioJanela >= 60000UL) {
    c.inicioJanela = agora;
    c.cont = 0;
  }

  if (agora - c.ultimo < INTERVALO_MIN_PEDIDOS_MS) {
    esperaMs = INTERVALO_MIN_PEDIDOS_MS - (agora - c.ultimo);
    return false;
  }

  if (c.cont >= LIMITE_PEDIDOS_POR_MIN) {
    esperaMs = 60000UL - (agora - c.inicioJanela);
    return false;
  }

  c.cont++;
  c.ultimo = agora;
  return true;
}

bool prepararPedido() {
  if (!autorizado()) return false;

  unsigned long espera = 0;

  if (!dentroDoLimite(server.client().remoteIP(), espera)) {
    unsigned long segundos = espera / 1000 + 1;
    server.sendHeader("Retry-After", String(segundos));
    server.send(429, "text/plain; charset=utf-8",
                "Calma! Muitas perguntas seguidas. Tente de novo em " + String(segundos) + " s.");
    return false;
  }

  return true;
}

// =====================================================
//  SERIAL (recuperar / trocar a senha)
// =====================================================

void verificarSerial() {
  while (Serial.available()) {
    char c = (char)Serial.read();

    if (c == '\n' || c == '\r') {
      linhaSerial.trim();

      if (linhaSerial == "senha") {
        mostrarSenhaNoSerial();
      } else if (linhaSerial == "novasenha") {
        senhaAcesso = gerarSenha(8);
        preferencias.putString("acesso", senhaAcesso);
        Serial.println("Nova senha gerada.");
        mostrarSenhaNoSerial();
      } else if (linhaSerial == "limpar_ia") {
        chaveAI = "";
        preferencias.remove("aikey");
        Serial.println("Chave da IA removida.");
      } else if (linhaSerial.length() > 0) {
        Serial.println("Comandos: senha | novasenha | limpar_ia");
      }

      linhaSerial = "";
    } else if (linhaSerial.length() < 40) {
      linhaSerial += c;
    }
  }
}

// =====================================================
//  PÁGINAS
// =====================================================

const char PAGINA_INICIO[] PROGMEM = R"rawliteral(
<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>TUXASSIST</title>

<style>
* { box-sizing: border-box; }

:root {
  --bg: #0b0f14; --card: #151b23; --borda: #26303c;
  --campo: #0d1218; --campoBorda: #303b49;
  --texto: #ffffff; --suave: #8e9baa;
  --destaque: #48e07b; --btnTexto: #07100a;
  --sec: #26303c; --secTexto: #ffffff;
}

:root[data-tema="claro"] {
  --bg: #f2f5f8; --card: #ffffff; --borda: #d8dee6;
  --campo: #f7f9fb; --campoBorda: #c3ccd6;
  --texto: #14202b; --suave: #5b6b7b;
  --destaque: #1f9d55; --btnTexto: #ffffff;
  --sec: #e3e9ef; --secTexto: #14202b;
}

body {
  margin: 0;
  min-height: 100vh;
  background: var(--bg);
  color: var(--texto);
  font-family: Arial, sans-serif;
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 20px;
}

.container { width: 100%; max-width: 700px; }
.header { text-align: center; margin-bottom: 25px; position: relative; }
.logo { font-size: 42px; }
h1 { margin: 8px 0; font-size: 30px; }
.status { color: var(--destaque); font-size: 14px; }

#btnTema {
  position: absolute; right: 0; top: 0;
  flex: none; width: 44px; padding: 10px;
  background: var(--sec); color: var(--secTexto);
}

.card {
  background: var(--card);
  border: 1px solid var(--borda);
  border-radius: 18px;
  padding: 20px;
  box-shadow: 0 10px 30px rgba(0,0,0,0.25);
}

textarea {
  width: 100%;
  min-height: 110px;
  resize: vertical;
  background: var(--campo);
  color: var(--texto);
  border: 1px solid var(--campoBorda);
  border-radius: 12px;
  padding: 15px;
  font-size: 17px;
  outline: none;
}

textarea:focus { border-color: var(--destaque); }

.buttons { display: flex; gap: 10px; margin-top: 12px; }

button {
  flex: 1;
  border: 0;
  border-radius: 12px;
  padding: 14px;
  font-size: 16px;
  font-weight: bold;
  cursor: pointer;
  background: var(--destaque);
  color: var(--btnTexto);
}

button:disabled { opacity: 0.5; cursor: wait; }
button.secondary { background: var(--sec); color: var(--secTexto); }
button.icone { flex: 0 0 56px; padding: 14px 0; }

.resposta {
  margin-top: 20px;
  background: var(--campo);
  border-radius: 12px;
  padding: 18px;
  min-height: 80px;
  white-space: pre-wrap;
  line-height: 1.6;
  font-size: 16px;
}

.ferramentas {
  display: flex; gap: 8px; align-items: center;
  flex-wrap: wrap; margin-top: 12px;
}

.ferramentas button { flex: none; padding: 8px 12px; font-size: 14px; }
.ferramentas label { font-size: 13px; color: var(--suave); }

.carregando { color: var(--destaque); }
.ip { text-align: center; margin-top: 15px; color: var(--suave); font-size: 13px; }
.ip a { color: var(--suave); }
</style>
</head>

<body>
<div class="container">

  <div class="header">
    <button id="btnTema" onclick="alternarTema()" title="Alternar tema">☀️</button>
    <div class="logo">🧠</div>
    <h1>TUXASSIST</h1>
    <div class="status">● ONLINE</div>
  </div>

  <div class="card">
    <textarea id="pergunta" maxlength="200" placeholder="Digite ou fale sua pergunta... (digite 'ajuda' para ver os comandos)"></textarea>

    <div class="buttons">
      <button id="btnVoz" class="secondary icone" onclick="falar()" title="Falar">🎤</button>
      <button id="btnCurta" onclick="perguntar(false)">Perguntar</button>
      <button id="btnLonga" class="secondary" onclick="perguntar(true)">Resposta longa</button>
    </div>

    <div id="resposta" class="resposta">Aguardando sua pergunta...</div>

    <div class="ferramentas">
      <button class="secondary" onclick="lerResposta()">🔊 Ouvir</button>
      <button class="secondary" onclick="pararLeitura()">⏹ Parar</button>
      <label><input type="checkbox" id="autoLer"> Ler automaticamente</label>
    </div>
  </div>

  <div class="ip">TUXASSIST • ESP32-C3 • IP: )rawliteral";

const char PAGINA_FIM[] PROGMEM = R"rawliteral( • <a href="/config">⚙ Configurações</a></div>

</div>

<script>
const $ = (id) => document.getElementById(id);
let ocupado = false;
let ultimaResposta = "";

/* ---------- Tema claro/escuro ---------- */
function aplicarTema(t) {
  document.documentElement.dataset.tema = t;
  $("btnTema").textContent = (t === "claro") ? "🌙" : "☀️";
}

function iniciarTema() {
  let t = null;
  try { t = localStorage.getItem("tema"); } catch (e) {}
  if (!t) {
    const claro = window.matchMedia && window.matchMedia("(prefers-color-scheme: light)").matches;
    t = claro ? "claro" : "escuro";
  }
  aplicarTema(t);
}

function alternarTema() {
  const novo = document.documentElement.dataset.tema === "claro" ? "escuro" : "claro";
  aplicarTema(novo);
  try { localStorage.setItem("tema", novo); } catch (e) {}
}

/* ---------- Perguntar (com streaming) ---------- */
function travar(estado) {
  ocupado = estado;
  $("btnCurta").disabled = estado;
  $("btnLonga").disabled = estado;
}

async function perguntar(longa) {
  if (ocupado) return;
  pararLeitura();

  const resposta = $("resposta");
  const pergunta = $("pergunta").value.trim();

  if (!pergunta) {
    resposta.textContent = "Digite uma pergunta.";
    return;
  }

  travar(true);
  resposta.innerHTML = '<span class="carregando">🔎 Pesquisando...</span>';
  ultimaResposta = "";

  const controle = new AbortController();
  const limite = setTimeout(() => controle.abort(), 90000);

  try {
    const rota = longa ? "/perguntar_longo" : "/perguntar";
    const r = await fetch(rota + "?texto=" + encodeURIComponent(pergunta), { signal: controle.signal });

    if (!r.body || !r.body.getReader) {
      ultimaResposta = await r.text();
      resposta.textContent = ultimaResposta;
    } else {
      const leitor = r.body.getReader();
      const decoder = new TextDecoder("utf-8");
      let primeiro = true;

      while (true) {
        const { done, value } = await leitor.read();
        if (done) break;
        primeiro = false;
        ultimaResposta += decoder.decode(value, { stream: true });
        resposta.textContent = ultimaResposta;
      }

      ultimaResposta += decoder.decode();
      resposta.textContent = primeiro ? "(sem resposta)" : ultimaResposta;
    }

    if ($("autoLer").checked && ultimaResposta) lerResposta();
  } catch (erro) {
    resposta.textContent = "❌ Não foi possível falar com o TUXASSIST.";
  } finally {
    clearTimeout(limite);
    travar(false);
  }
}

/* ---------- Ler resposta em voz alta ---------- */
function lerResposta() {
  if (!("speechSynthesis" in window) || !ultimaResposta) return;
  window.speechSynthesis.cancel();

  const fala = new SpeechSynthesisUtterance(ultimaResposta);
  fala.lang = "pt-BR";

  const voz = window.speechSynthesis.getVoices().find(
    (v) => v.lang && v.lang.toLowerCase().startsWith("pt-br")
  );
  if (voz) fala.voice = voz;

  window.speechSynthesis.speak(fala);
}

function pararLeitura() {
  if ("speechSynthesis" in window) window.speechSynthesis.cancel();
}

/* ---------- Entrada por voz ---------- */
const Reconhecimento = window.SpeechRecognition || window.webkitSpeechRecognition;
let reconhecedor = null;
let ouvindo = false;

function falar() {
  const resposta = $("resposta");

  if (!Reconhecimento) {
    resposta.textContent = "Este navegador não suporta reconhecimento de voz. Use o microfone do teclado.";
    return;
  }

  if (ouvindo && reconhecedor) {
    reconhecedor.stop();
    return;
  }

  pararLeitura();

  reconhecedor = new Reconhecimento();
  reconhecedor.lang = "pt-BR";
  reconhecedor.interimResults = true;
  reconhecedor.continuous = false;

  reconhecedor.onstart = () => { ouvindo = true; $("btnVoz").textContent = "🛑"; };
  reconhecedor.onend = () => { ouvindo = false; $("btnVoz").textContent = "🎤"; };

  reconhecedor.onresult = (ev) => {
    let texto = "";
    let final = false;
    for (let i = 0; i < ev.results.length; i++) {
      texto += ev.results[i][0].transcript;
      if (ev.results[i].isFinal) final = true;
    }
    $("pergunta").value = texto;
    if (final) perguntar(false);
  };

  reconhecedor.onerror = (ev) => {
    if (ev.error === "not-allowed" || ev.error === "service-not-allowed") {
      resposta.textContent =
        "O navegador bloqueou o microfone neste endereço (http). " +
        "Use o microfone do teclado do celular, ou libere o endereço em chrome://flags " +
        "(Insecure origins treated as secure).";
    } else if (ev.error !== "aborted" && ev.error !== "no-speech") {
      resposta.textContent = "Erro no microfone: " + ev.error;
    }
  };

  reconhecedor.start();
}

/* ---------- Atalhos ---------- */
$("pergunta").addEventListener("keydown", function (event) {
  if (event.key === "Enter" && !event.shiftKey) {
    event.preventDefault();
    perguntar(false);
  }
});

iniciarTema();
if ("speechSynthesis" in window) window.speechSynthesis.getVoices();
</script>

</body>
</html>
)rawliteral";

const char PAGINA_WIFI[] PROGMEM = R"rawliteral(
<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>TUXASSIST - Wi-Fi</title>
<style>
* { box-sizing: border-box; }
body { margin: 0; padding: 20px; background: #0b0f14; color: #fff; font-family: Arial, sans-serif; }
.card { max-width: 420px; margin: 40px auto; background: #151b23; border: 1px solid #26303c; border-radius: 18px; padding: 24px; }
h1 { margin-top: 0; font-size: 24px; }
label { display: block; margin: 14px 0 6px; font-size: 14px; color: #8e9baa; }
input { width: 100%; padding: 12px; font-size: 16px; border-radius: 10px; border: 1px solid #303b49; background: #0d1218; color: #fff; }
button { width: 100%; margin-top: 20px; padding: 14px; font-size: 16px; font-weight: bold; border: 0; border-radius: 12px; background: #48e07b; color: #07100a; }
</style>
</head>
<body>
<div class="card">
  <h1>🧠 TUXASSIST - Wi-Fi</h1>
  <form method="POST" action="/wifi">
    <label for="ssid">Nome da rede (SSID)</label>
    <input id="ssid" name="ssid" maxlength="32" required>
    <label for="senha">Senha</label>
    <input id="senha" name="senha" type="password" maxlength="63">
    <button type="submit">Salvar e reiniciar</button>
  </form>
</div>
</body>
</html>
)rawliteral";

const char PAGINA_CONFIG_INI[] PROGMEM = R"rawliteral(
<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>TUXASSIST - Configurações</title>
<style>
* { box-sizing: border-box; }
body { margin: 0; padding: 20px; background: #0b0f14; color: #fff; font-family: Arial, sans-serif; }
.card { max-width: 460px; margin: 30px auto; background: #151b23; border: 1px solid #26303c; border-radius: 18px; padding: 24px; }
h1 { margin-top: 0; font-size: 22px; }
.info { background: #0d1218; border-radius: 10px; padding: 14px; line-height: 1.7; font-size: 14px; }
label { display: block; margin: 16px 0 6px; font-size: 14px; color: #8e9baa; }
input[type=text], input[type=password] { width: 100%; padding: 12px; font-size: 16px; border-radius: 10px; border: 1px solid #303b49; background: #0d1218; color: #fff; }
button { width: 100%; margin-top: 20px; padding: 14px; font-size: 16px; font-weight: bold; border: 0; border-radius: 12px; background: #48e07b; color: #07100a; }
a { color: #8e9baa; display: block; text-align: center; margin-top: 16px; }
.nota { font-size: 12px; color: #8e9baa; margin-top: 6px; }
</style>
</head>
<body>
<div class="card">
  <h1>⚙ TUXASSIST - Configurações</h1>
  <div class="info">)rawliteral";

const char PAGINA_CONFIG_FIM[] PROGMEM = R"rawliteral(</div>

  <form method="POST" action="/config">
    <label for="aikey">Chave da API da IA (Groq)</label>
    <input id="aikey" name="aikey" type="password" maxlength="200" placeholder="deixe vazio para manter a atual" autocomplete="off">
    <div class="nota">Chave grátis em console.groq.com/keys</div>
    <label><input type="checkbox" name="apagarchave" value="1"> Apagar a chave salva</label>

    <label for="acesso">Nova senha de acesso (6 a 32 caracteres)</label>
    <input id="acesso" name="acesso" type="password" maxlength="32" placeholder="deixe vazio para manter a atual" autocomplete="new-password">
    <div class="nota">Depois de trocar, o navegador vai pedir a senha nova.</div>

    <button type="submit">Salvar</button>
  </form>

  <a href="/">← Voltar</a>
</div>
</body>
</html>
)rawliteral";

// =====================================================
//  HANDLERS
// =====================================================

void handleRoot() {
  if (modoConfig && WiFi.status() != WL_CONNECTED) {
    server.sendHeader("Location", "/wifi");
    server.send(302, "text/plain", "");
    return;
  }

  if (!autorizado()) return;

  server.setContentLength(CONTENT_LENGTH_UNKNOWN);
  server.send(200, "text/html; charset=utf-8", "");
  server.sendContent_P(PAGINA_INICIO);
  server.sendContent(WiFi.localIP().toString());
  server.sendContent_P(PAGINA_FIM);
  server.sendContent("");
}

void handleStatus() {
  if (!autorizado()) return;

  String resposta;

  resposta += "TUXASSIST STATUS\n";
  resposta += "Wi-Fi: ";
  resposta += (WiFi.status() == WL_CONNECTED) ? "conectado\n" : "desconectado\n";
  resposta += "SSID: ";
  resposta += ssid;
  resposta += "\nIP: ";
  resposta += WiFi.localIP().toString();
  resposta += "\nRSSI: ";
  resposta += String(WiFi.RSSI());
  resposta += " dBm\n";
  resposta += "IA: ";
  resposta += (chaveAI.length() > 0) ? "configurada\n" : "sem chave\n";
  resposta += "Modelo: ";
  resposta += AI_MODELOS[0];
  resposta += "\nHTTPS: ";
  resposta += VALIDANDO_CERTIFICADO ? "certificado validado\n" : "SEM validacao de certificado\n";
  resposta += "Hora: ";
  resposta += horaValida() ? "sincronizada\n" : "nao sincronizada\n";
  resposta += "Memoria livre: ";
  resposta += String(ESP.getFreeHeap());
  resposta += " bytes\n";
  resposta += "Tempo ligado: ";
  resposta += String(millis() / 1000);
  resposta += " s\n";

  server.send(200, "text/plain; charset=utf-8", resposta);
}

void handleConfigForm() {
  if (!autorizado()) return;

  String info;

  if (server.hasArg("salvo")) {
    info += "✔ Alterações salvas.<br><br>";
  }

  info += "Chave da IA: ";
  info += (chaveAI.length() > 0) ? "configurada ✔" : "não configurada ✖";
  info += "<br>Modelo: ";
  info += AI_MODELOS[0];
  info += "<br>HTTPS: ";
  info += VALIDANDO_CERTIFICADO ? "certificado validado ✔" : "sem validação de certificado ⚠";
  info += "<br>Hora: ";
  info += horaValida() ? "sincronizada ✔" : "não sincronizada ✖";

  server.setContentLength(CONTENT_LENGTH_UNKNOWN);
  server.send(200, "text/html; charset=utf-8", "");
  server.sendContent_P(PAGINA_CONFIG_INI);
  server.sendContent(info);
  server.sendContent_P(PAGINA_CONFIG_FIM);
  server.sendContent("");
}

void handleConfigSalvar() {
  if (!autorizado()) return;

  String novaSenha = server.arg("acesso");
  String novaChave = server.arg("aikey");
  novaSenha.trim();
  novaChave.trim();

  if (novaSenha.length() > 0 && (novaSenha.length() < 6 || novaSenha.length() > 32)) {
    server.send(400, "text/plain; charset=utf-8", "Senha de acesso: use de 6 a 32 caracteres.");
    return;
  }

  if (novaChave.length() > 200) {
    server.send(400, "text/plain; charset=utf-8", "Chave da API muito longa.");
    return;
  }

  if (server.hasArg("apagarchave")) {
    chaveAI = "";
    preferencias.remove("aikey");
  }

  if (novaChave.length() > 0) {
    chaveAI = novaChave;
    preferencias.putString("aikey", chaveAI);
  }

  if (novaSenha.length() > 0) {
    senhaAcesso = novaSenha;
    preferencias.putString("acesso", senhaAcesso);
  }

  server.sendHeader("Location", "/config?salvo=1");
  server.send(303, "text/plain", "");
}

void handleWifiForm() {
  if (!modoConfig) {
    server.send(403, "text/plain; charset=utf-8", "Configuracao de Wi-Fi indisponivel.");
    return;
  }

  server.send_P(200, "text/html; charset=utf-8", PAGINA_WIFI);
}

void handleWifiSalvar() {
  if (!modoConfig) {
    server.send(403, "text/plain; charset=utf-8", "Configuracao de Wi-Fi indisponivel.");
    return;
  }

  String novoSsid = server.arg("ssid");
  String novaSenha = server.arg("senha");
  novoSsid.trim();

  if (novoSsid.length() == 0 || novoSsid.length() > 32) {
    server.send(400, "text/plain; charset=utf-8", "SSID invalido (1 a 32 caracteres).");
    return;
  }

  if (novaSenha.length() != 0 && (novaSenha.length() < 8 || novaSenha.length() > 63)) {
    server.send(400, "text/plain; charset=utf-8", "Senha invalida (8 a 63 caracteres, ou vazia para rede aberta).");
    return;
  }

  preferencias.putString("ssid", novoSsid);
  preferencias.putString("senha", novaSenha);

  server.send(200, "text/plain; charset=utf-8", "Wi-Fi salvo. Reiniciando...");

  delay(1500);
  ESP.restart();
}

void responderPergunta(String pergunta, bool longa) {
  pergunta.trim();

  if (pergunta.length() == 0) {
    server.send(400, "text/plain; charset=utf-8", "Erro: pergunta vazia.");
    return;
  }

  if (pergunta.length() > MAX_PERGUNTA) {
    server.send(400, "text/plain; charset=utf-8", "Erro: pergunta muito longa (maximo 200 caracteres).");
    return;
  }

  Serial.println();
  Serial.println("==============================");
  Serial.println("PERGUNTA:");
  Serial.println(pergunta);
  Serial.println("==============================");

  // 1) Comandos especiais
  String texto;

  if (tratarComando(pergunta, texto)) {
    Serial.println("Fonte: comando");
    Serial.println(texto);
    enviarCompleto(texto);
    return;
  }

  if (WiFi.status() != WL_CONNECTED) {
    server.send(503, "text/plain; charset=utf-8", "Sem conexao com a internet no momento.");
    return;
  }

  // 2) IA da Groq (streaming), tentando os modelos da lista
  erroIA = "";

  if (perguntarIA(pergunta, longa)) {
    Serial.println("Fonte: IA");
    finalizarResposta();
    return;
  }

  // 3) Wikipedia (reserva)
  texto = pegarResumoWikipedia(pergunta, longa ? 10 : 3);

  if (texto.length() > 0) {
    Serial.println("Fonte: Wikipedia");
  } else {
    // 4) DuckDuckGo (reserva)
    Serial.println("Wikipedia nao encontrou.");
    texto = pesquisarDuckDuckGo(pergunta);

    if (texto.length() > 0) {
      Serial.println("Fonte: DuckDuckGo");
    }
  }

  if (texto.length() == 0) {
    texto = "Nao encontrei uma resposta para essa pergunta.";
  }

  texto = formatarResposta(texto, LARGURA_TEXTO);

  if (chaveAI.length() == 0) {
    texto += "\n\n(Dica: configure a chave da IA em /config para respostas melhores.)";
  } else if (erroIA.length() > 0) {
    texto += "\n\n(IA indisponível: " + erroIA + ")";
  }

  Serial.println();
  Serial.println("RESPOSTA:");
  Serial.println(texto);
  Serial.println("==============================");

  enviarCompleto(texto);
}

void handlePerguntar() {
  if (!server.hasArg("texto")) {
    server.send(400, "text/plain; charset=utf-8", "Erro: use /perguntar?texto=Sua pergunta");
    return;
  }

  if (!prepararPedido()) return;

  responderPergunta(server.arg("texto"), false);
}

void handlePerguntarLongo() {
  if (!server.hasArg("texto")) {
    server.send(400, "text/plain; charset=utf-8", "Erro: use /perguntar_longo?texto=Sua pergunta");
    return;
  }

  if (!prepararPedido()) return;

  responderPergunta(server.arg("texto"), true);
}

void handleNaoEncontrado() {
  server.send(404, "text/plain; charset=utf-8", "Pagina nao encontrada.");
}

// =====================================================
//  SETUP / LOOP
// =====================================================

void setup() {
  Serial.begin(115200);
  delay(1000);

  Serial.println();
  Serial.println("================================");
  Serial.println("       ESP32-C3 TUXASSIST");
  Serial.println("================================");

  if (!VALIDANDO_CERTIFICADO) {
    Serial.println("AVISO: HTTPS sem validacao de certificado (instale a biblioteca ESP32CertBundle).");
  }

  preferencias.begin("wifi", false);

  ssid = preferencias.getString("ssid", "");
  senha = preferencias.getString("senha", "");
  senhaAcesso = preferencias.getString("acesso", "");
  chaveAI = preferencias.getString("aikey", "");

  if (ssid.length() > 0) {
    Serial.println("Wi-Fi encontrado na memoria.");
    conectarWiFi();
  } else {
    Serial.println("Nenhum Wi-Fi salvo.");
  }

  if (WiFi.status() == WL_CONNECTED) {
    estavaConectado = true;
    aoConectar();
  } else {
    iniciarModoConfig();
  }

  // Senha de acesso: usa a SENHA_INICIAL do código (usuário: admin).
  if (RESETAR_SENHA_NO_BOOT || senhaAcesso.length() == 0) {
    senhaAcesso = SENHA_INICIAL;
    preferencias.putString("acesso", senhaAcesso);
    Serial.println("Senha de acesso definida para a senha inicial.");
  }

  if (chaveAI.length() == 0) {
    Serial.println("IA sem chave: abra /config e cole a chave da Groq.");
  }

  const char *cabecalhos[] = { "Authorization" };
  server.collectHeaders(cabecalhos, 1);

  server.on("/", HTTP_GET, handleRoot);
  server.on("/status", HTTP_GET, handleStatus);
  server.on("/perguntar", HTTP_GET, handlePerguntar);
  server.on("/perguntar_longo", HTTP_GET, handlePerguntarLongo);
  server.on("/config", HTTP_GET, handleConfigForm);
  server.on("/config", HTTP_POST, handleConfigSalvar);
  server.on("/wifi", HTTP_GET, handleWifiForm);
  server.on("/wifi", HTTP_POST, handleWifiSalvar);
  server.onNotFound(handleNaoEncontrado);

  server.begin();

  Serial.println();
  Serial.println("Servidor HTTP iniciado!");
  Serial.println("TUXASSIST ONLINE!");
}

void loop() {
  server.handleClient();
  verificarDescoberta();
  verificarWiFi();
  verificarSerial();
  delay(2);
}