#!/bin/bash

# Script para gerar certificados assinados por uma CA interna
# Requisitos: pacote 'mailutils' (ou 'mail' compatível)

BLUE="\033[0;36m"
RED="\033[0;31m"
NC="\033[0m"

if [[ "$1" == "-h" || "$1" == "--help" ]]; then
    echo -e "\n${BLUE}Uso:${NC}"
    echo "  ./gera_certificado.sh -CSR <dominio> [-A <1|2>] [-N <nome>] [-W true] [-M email] [-D dias]"
    echo "  ./gera_certificado.sh -F requisicoes.txt"
    echo -e "\n${BLUE}Parâmetros:${NC}"
    echo "  -CSR     Nome base do CSR (obrigatório)"
    echo "  -A       Ambiente: 1 = HML, 2 = PRD (padrão: 2)"
    echo "  -N       Nome do certificado gerado (padrão: mesmo do CSR)"
    echo "  -W       true = gera como wildcard (*.dominio.com)"
    echo "  -M       E-mail de destino para envio dos arquivos gerados"
    echo "  -D       Dias de validade do certificado (padrão: 365 PRD, 30 HML)"
    echo "  -F       Arquivo de entrada para processamento em lote"
    echo -e "\n${BLUE}Exemplo:${NC}"
    echo "  ./gera_certificado.sh -CSR exemplo.com -A 2 -W true -M admin@empresa.com -D 730"
    echo "  ./gera_certificado.sh -F lista.txt"
    exit 0
fi

mkdir -p logs

registrar_log() {
    local log_file="logs/cert-${1}.log"
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $2" >> "$log_file"
}

enviar_email() {
    local email="$1"
    local nome="$2"
    local path="$3"
    local crt_file="$path/$nome.crt"
    local ca_file="$path/rootca.pem"

    if [ -f "$crt_file" ] && [ -f "$ca_file" ]; then
        echo "Segue em anexo o certificado '$nome.crt'" | \
        mail -s "Certificado Gerado: $nome" -a "$crt_file" -a "$ca_file" "$email"
        registrar_log "$nome" "E-mail enviado para $email com certificado."
    else
        registrar_log "$nome" "Erro ao enviar e-mail: arquivos não encontrados."
    fi
}

processar_certificado() {
    local op="$1"
    local CSR="$2"
    local CRT="$3"
    local WILDCARD="$4"
    local EMAIL="$5"
    local DIAS="$6"
    local pathcert="/home/kali/Desktop/certificados"
    local usrpath="$(pwd)"
    local OUTDIR="$usrpath/$CSR"

    mkdir -p "$OUTDIR"

    if [ ! -f "$usrpath/$CSR.csr" ]; then
        echo -e "${RED}[ERRO] CSR '$CSR.csr' não encontrado em $usrpath${NC}"
        registrar_log "$CSR" "Erro: CSR não encontrado."
        return 1
    fi

    if [ "$WILDCARD" == "true" ]; then
        CN="*.$CSR"
        ALT="DNS.1 = *.$CSR"
    else
        CN="$CSR"
        ALT="DNS.1 = $CSR"
    fi

    cat <<EOF > v3_req.conf
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_req
prompt = no

[req_distinguished_name]
C = BR
ST = Sao Paulo
L = Sao Paulo
O = Meu Dominio
OU = TI
CN = $CN

[v3_req]
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName = @alt_names

[alt_names]
$ALT
EOF

    # Define dias se não for informado
    if [ -z "$DIAS" ]; then
        if [ "$op" == "2" ]; then
            DIAS=365
        else
            DIAS=30
        fi
    fi

    if [ "$op" == "2" ]; then
        if [ ! -f "$pathcert/prd-ca-cert.pem" ] || [ ! -f "$pathcert/prd-ca-key.key" ]; then
            echo -e "${RED}[ERRO] Arquivos da CA PRD não encontrados.${NC}"
            registrar_log "$CSR" "Erro: CA PRD ausente."
            return 1
        fi
        openssl x509 -extfile v3_req.conf -req -in "$usrpath/$CSR.csr" \
            -CA "$pathcert/prd-ca-cert.pem" -CAkey "$pathcert/prd-ca-key.key" -CAcreateserial \
            -out "$OUTDIR/$CRT.crt" -days "$DIAS" -sha256 -extensions v3_req
        cp "$pathcert/prd-ca-cert.pem" "$OUTDIR/rootca.pem"

    elif [ "$op" == "1" ]; then
        if [ ! -f "$pathcert/hml-ca-cert.pem" ] || [ ! -f "$pathcert/hml-ca-key.key" ]; then
            echo -e "${RED}[ERRO] Arquivos da CA HML não encontrados.${NC}"
            registrar_log "$CSR" "Erro: CA HML ausente."
            return 1
        fi
        openssl x509 -extfile v3_req.conf -req -in "$usrpath/$CSR.csr" \
            -CA "$pathcert/hml-ca-cert.pem" -CAkey "$pathcert/hml-ca-key.key" -CAcreateserial \
            -out "$OUTDIR/$CRT.crt" -days "$DIAS" -sha256 -extensions v3_req
        cp "$pathcert/hml-ca-cert.pem" "$OUTDIR/rootca.pem"

    else
        echo -e "${RED}[ERRO] Ambiente inválido. Use 1 (HML) ou 2 (PRD).${NC}"
        registrar_log "$CSR" "Erro: ambiente inválido."
        return 1
    fi

    chmod 755 "$OUTDIR/rootca.pem"
    echo -e "${BLUE}[OK] Certificado gerado: ${NC}${RED}$OUTDIR/$CRT.crt${NC}"
    registrar_log "$CSR" "Certificado $CRT.crt gerado com sucesso."

    [ -n "$EMAIL" ] && enviar_email "$EMAIL" "$CRT" "$OUTDIR"
}

# Execução em lote
if [ "$1" == "-F" ]; then
    LISTA="$2"
    if [ ! -f "$LISTA" ]; then
        echo -e "${RED}Arquivo '$LISTA' não encontrado.${NC}"
        exit 1
    fi

    while IFS= read -r linha || [[ -n "$linha" ]]; do
        eval set -- $linha
        op="2"; CSR=""; CRT=""; WILDCARD="false"; EMAIL=""; DIAS=""
        while [[ $# -gt 0 ]]; do
            case "$1" in
                -A) op="$2"; shift 2 ;;
                -CSR) CSR="$2"; shift 2 ;;
                -N) CRT="$2"; shift 2 ;;
                -W) WILDCARD="$2"; shift 2 ;;
                -M) EMAIL="$2"; shift 2 ;;
                -D) DIAS="$2"; shift 2 ;;
                *) echo -e "${RED}[WARN] Flag inválida: $1${NC}"; shift ;;
            esac
        done
        [ -z "$CRT" ] && CRT="$CSR"
        echo -e "${BLUE}Processando: -A $op -CSR $CSR -N $CRT -W $WILDCARD -M $EMAIL -D $DIAS${NC}"
        processar_certificado "$op" "$CSR" "$CRT" "$WILDCARD" "$EMAIL" "$DIAS"
        echo
    done < "$LISTA"
    exit 0
fi

# Execução manual
op="2"; CSR=""; CRT=""; WILDCARD="false"; EMAIL=""; DIAS=""

while [[ $# -gt 0 ]]; do
    case "$1" in
        -A) op="$2"; shift 2 ;;
        -CSR) CSR="$2"; shift 2 ;;
        -N) CRT="$2"; shift 2 ;;
        -W) WILDCARD="$2"; shift 2 ;;
        -M) EMAIL="$2"; shift 2 ;;
        -D) DIAS="$2"; shift 2 ;;
        -h|--help)
            echo -e "${BLUE}Use -h ou --help fora do modo de execução.${NC}"
            exit 0 ;;
        *)
            echo -e "${RED}Parâmetro inválido: $1${NC}"
            echo -e "Use -h ou --help para ajuda."
            exit 1 ;;
    esac
done

if [[ -z "$CSR" ]]; then
    echo -e "${RED}Parâmetro obrigatório: -CSR <nome>${NC}"
    exit 1
fi

[ -z "$CRT" ] && CRT="$CSR"
processar_certificado "$op" "$CSR" "$CRT" "$WILDCARD" "$EMAIL" "$DIAS"
