"""
Gerador de dados sinteticos anonimizados - MVP Pipeline de Agendamentos B2B

Gera 4 arquivos brutos (camada Bronze) simulando os 4 sistemas de origem,
com o MESMO TIPO de sujeira/inconsistencia dos dados reais, mas 100%
ficticios: nenhum contrato, cliente, endereco ou valor real e usado.

Uso:
    python gerar_dados_sinteticos.py

Gera em ./bronze/:
    sistema_a.csv   (CRM padrao)
    sistema_b.xlsx  (CRM regional 1, aba 'data')
    sistema_c.xlsx  (CRM regional 2, nome de arquivo com sufixo variavel)
    sistema_d.xlsx  (planilha de varejo)
"""

import random
import string
from datetime import datetime, timedelta
from pathlib import Path

import pandas as pd

random.seed(42)

OUT_DIR = Path(__file__).parent / "bronze"
OUT_DIR.mkdir(exist_ok=True)

N_CONTRATOS = 400

# --- dados de referencia ficticios -----------------------------------------

TERRITORIOS_CANONICOS = [
    "SP SP METROPOLITANO",
    "SP SP CAMPINAS",
    "RJ RJ CAPITAL",
    "MG BH METROPOLITANO",
    "RS POA METROPOLITANO",
]

# grafias divergentes de propósito, pra simular a sujeira real entre sistemas
VARIANTES_TERRITORIO = {
    "SP SP METROPOLITANO": ["SP SP METROPOLITANO", "SP - Metropolitana", "sp metropolitano", "SAO PAULO METRO"],
    "SP SP CAMPINAS": ["SP SP CAMPINAS", "Campinas-SP", "sp campinas"],
    "RJ RJ CAPITAL": ["RJ RJ CAPITAL", "Rio de Janeiro Capital", "rj capital"],
    "MG BH METROPOLITANO": ["MG BH METROPOLITANO", "Belo Horizonte Metro", "mg bh metro"],
    "RS POA METROPOLITANO": ["RS POA METROPOLITANO", "Porto Alegre Metro", "rs poa metro"],
}

TIPOS_ATIVIDADE = ["INSTALACAO", "VISTORIA"]

STATUS_POR_SISTEMA = {
    "A": {"agendado": "AGENDADO INFRA", "atrasado": "PENDENTE POS VENDAS", "oportunidade": "AGENDAR INFRA"},
    "B": {"agendado": "AGENDADO", "atrasado": "EM FOLLOW-UP", "oportunidade": "SEM DATA"},
    "C": {"agendado": "OS AGENDADA", "atrasado": "ABERTA", "oportunidade": "AGUARDANDO AGENDA"},
    "D": {"agendado": "AGENDADO", "atrasado": "PENDENTE", "oportunidade": "PENDENTE"},
}

# raizes de CNPJ ficticias para simular contratos replicados entre sistemas
CNPJ_RAIZ_REPLICADO = {"99887766", "11223344"}


def gerar_cnpj_falso(replicado=False):
    raiz = random.choice(list(CNPJ_RAIZ_REPLICADO)) if replicado else "".join(random.choices(string.digits, k=8))
    return raiz + "".join(random.choices(string.digits, k=6))  # 14 digitos = PJ


def gerar_cpf_falso():
    return "".join(random.choices(string.digits, k=11))  # 11 digitos = PF


def data_aleatoria(dias_janela=45, permitir_nula=False, chance_nula=0.15):
    if permitir_nula and random.random() < chance_nula:
        return None
    delta = random.randint(-10, dias_janela)
    return (datetime(2026, 8, 1) + timedelta(days=delta)).strftime("%Y-%m-%d")


# --- gera a "verdade" por tras dos 4 sistemas -------------------------------

registros = []
for i in range(N_CONTRATOS):
    contrato_id = 3_000_000 + i
    territorio_canonico = random.choice(TERRITORIOS_CANONICOS)
    is_replicado = random.random() < 0.05  # ~5% dos contratos replicados entre sistemas
    sistema_origem = random.choice(["A", "B", "C", "D"])
    tipo_atividade = random.choice(TIPOS_ATIVIDADE)
    registros.append(
        {
            "contrato_id": contrato_id,
            "territorio_canonico": territorio_canonico,
            "sistema_origem": sistema_origem,
            "tipo_atividade": tipo_atividade,
            "is_replicado": is_replicado,
            "valor_recorrente": round(random.uniform(150, 3500), 2),
        }
    )

df_verdade = pd.DataFrame(registros)

# --- Sistema A: CRM padrao (CSV) --------------------------------------------

linhas_a = []
for _, r in df_verdade.iterrows():
    if r["sistema_origem"] != "A" and not r["is_replicado"]:
        continue
    territorio_txt = random.choice(VARIANTES_TERRITORIO[r["territorio_canonico"]])
    situacao = random.choices(["agendado", "atrasado", "oportunidade"], weights=[0.5, 0.2, 0.3])[0]
    linhas_a.append(
        {
            "NUM_CONTRATO": r["contrato_id"],
            "CNPJ_CPF": gerar_cnpj_falso(replicado=r["is_replicado"]),
            "TERRITORIO": territorio_txt,
            "TIPO_PENDENCIA": STATUS_POR_SISTEMA["A"][situacao],
            "DATA_AGENDAMENTO": data_aleatoria(permitir_nula=(situacao == "oportunidade"), chance_nula=0.6),
            "VALOR_RECORRENTE": r["valor_recorrente"],
            "TIPO_ATIVIDADE": "AGENDADO INFRA" if r["tipo_atividade"] == "INSTALACAO" else "AGENDADO VISTORIA",
        }
    )
pd.DataFrame(linhas_a).to_csv(OUT_DIR / "sistema_a.csv", index=False, sep=";", encoding="utf-8")

# --- Sistema B: CRM regional 1 (XLSX, aba 'data') ---------------------------

linhas_b = []
for _, r in df_verdade.iterrows():
    if r["sistema_origem"] != "B":
        continue
    # mistura PF e PJ de proposito, igual ao caso real
    documento = gerar_cpf_falso() if random.random() < 0.4 else gerar_cnpj_falso()
    territorio_txt = random.choice(VARIANTES_TERRITORIO[r["territorio_canonico"]])
    situacao = random.choices(["agendado", "atrasado", "oportunidade"], weights=[0.4, 0.25, 0.35])[0]
    linhas_b.append(
        {
            "ID_CONTRATO": r["contrato_id"],
            "DOCUMENTO": documento,
            "CLUSTER": territorio_txt,
            "SERVICO_ABERTURA": "INSTALACAO FIBRA" if r["tipo_atividade"] == "INSTALACAO" else "VISTORIA FIBRA",
            "STATUS": STATUS_POR_SISTEMA["B"][situacao],
            "DATA_ABERTURA": data_aleatoria(permitir_nula=True, chance_nula=0.4),
            "PERIODO": random.choice(["MANHA", "TARDE", None]),
        }
    )
with pd.ExcelWriter(OUT_DIR / "sistema_b.xlsx", engine="openpyxl") as writer:
    pd.DataFrame(linhas_b).to_excel(writer, sheet_name="data", index=False)

# --- Sistema C: CRM regional 2 (XLSX, nome de arquivo variavel) -------------

linhas_c = []
for _, r in df_verdade.iterrows():
    if r["sistema_origem"] != "C" and not r["is_replicado"]:
        continue
    territorio_txt = random.choice(VARIANTES_TERRITORIO[r["territorio_canonico"]])
    situacao = random.choices(["agendado", "atrasado", "oportunidade"], weights=[0.45, 0.2, 0.35])[0]
    linhas_c.append(
        {
            "Contrato": r["contrato_id"],
            "CNPJ": gerar_cnpj_falso(replicado=r["is_replicado"]),
            "Territorio_Atend": territorio_txt,
            "Tipo Atendimento": "INSTALACAO (OS)" if r["tipo_atividade"] == "INSTALACAO" else "VISTORIA (OS)",
            "Servico": "INSTALACAO" if r["tipo_atividade"] == "INSTALACAO" else "VISTORIA",
            "Tecnologia": "FIBRA",
            "Situacao_OS": STATUS_POR_SISTEMA["C"][situacao],
            "Data_Agenda": data_aleatoria(permitir_nula=True, chance_nula=0.35),
        }
    )
# nome de arquivo com sufixo variavel, simulando exports com nome dinamico
sufixo = random.randint(1000, 9999)
pd.DataFrame(linhas_c).to_excel(OUT_DIR / f"sistema_c_relatorio({sufixo}).xlsx", index=False)

# --- Sistema D: planilha de varejo (XLSX) -----------------------------------

linhas_d = []
for _, r in df_verdade.iterrows():
    if r["sistema_origem"] != "D":
        continue
    territorio_txt = random.choice(VARIANTES_TERRITORIO[r["territorio_canonico"]])
    situacao = random.choices(["agendado", "atrasado", "oportunidade"], weights=[0.3, 0.3, 0.4])[0]
    uf = territorio_txt[:2].upper() if territorio_txt[:2].isalpha() else "SP"
    linhas_d.append(
        {
            "Numero_Contrato": r["contrato_id"],
            "UF": uf,
            "Cluster_Varejo": territorio_txt,
            "Status": STATUS_POR_SISTEMA["D"][situacao],
            "Data_Agendamento": data_aleatoria(permitir_nula=True, chance_nula=0.5),
        }
    )
pd.DataFrame(linhas_d).to_excel(OUT_DIR / "sistema_d.xlsx", index=False)

print(f"Arquivos gerados em: {OUT_DIR.resolve()}")
for f in sorted(OUT_DIR.iterdir()):
    print(" -", f.name)
