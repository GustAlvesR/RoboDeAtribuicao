import pandas as pd
from playwright.sync_api import sync_playwright
import time
import os

# =========================
# 1. CONFIGURAÇÃO DO CSV
# =========================
nome_arquivo = "atribuicaomeli.csv"

if not os.path.exists(nome_arquivo):
    print(f"❌ ERRO: Arquivo '{nome_arquivo}' não encontrado!")
    exit()

try:
    df = pd.read_csv(nome_arquivo, skiprows=2, header=None, sep=None, engine='python', encoding='utf-8-sig')
    df = df.iloc[:, [0, 3, 4]] 
    df.columns = ['ROTA', 'PLACA', 'MOTORISTA']
    df = df.dropna(subset=['ROTA'])
    dados_planilha = df.set_index('ROTA').T.to_dict()
    print(f"✅ Planilha carregada. {len(dados_planilha)} rotas mapeadas.")
except Exception as e:
    print(f"❌ Erro ao ler CSV: {e}")
    exit()

# =========================
# 2. AUTOMAÇÃO COM PAGINAÇÃO
# =========================
with sync_playwright() as p:
    browser = p.chromium.launch(headless=False, channel="chrome")
    page = browser.new_page()
    page.goto("https://envios.adminml.com/logistics/fm/planning/routes-planned")

    print("\n👋 MODO FLUXO: Processa a página e avança no 'Seguinte'.")
    input("👉 Faça login, filtre e pressione ENTER aqui...")

    while True:
        # 🎯 PASSO 1: Carregar os cards da página atual
        print("⏬ Carregando cards da página...")
        page.keyboard.press("End") # Vai até o fim para carregar o Lazy Load
        time.sleep(2)
        
        cards = page.locator(".andes-card, .route-card").all()
        encontrou_na_pagina = False

        # 🎯 PASSO 2: Percorrer os cards da página
        for card in cards:
            texto_card = card.inner_text()
            rota_id = next((r for r in dados_planilha.keys() if str(r) in texto_card), None)

            if rota_id:
                encontrou_na_pagina = True
                moto = str(dados_planilha[rota_id]['MOTORISTA']).strip()
                placa = str(dados_planilha[rota_id]['PLACA']).strip()

                print(f"🚀 Rota encontrada: {rota_id}")
                
                try:
                    card.scroll_into_view_if_needed()
                    card.locator("button[aria-label*='menu'], .andes-button--show-more").first.click()
                    page.locator("button.andes-list__item-actionable, .andes-list__item-primary").first.click()
                    
                    page.wait_for_selector(".andes-modal__content", timeout=10000)
                    
                    # Preenchimento (Lógica Rápida)
                    page.keyboard.press("Tab"); page.keyboard.press("Enter"); time.sleep(0.5)
                    page.keyboard.type(moto, delay=80); time.sleep(3.5)
                    page.keyboard.press("ArrowDown"); page.keyboard.press("Enter"); time.sleep(1.0)

                    page.keyboard.press("Tab"); page.keyboard.press("Enter"); time.sleep(0.5)
                    page.keyboard.type(placa, delay=80); time.sleep(3.0)
                    page.keyboard.press("ArrowDown"); page.keyboard.press("Enter"); time.sleep(1.0)

                    # Salvar
                    page.keyboard.press("Tab"); page.keyboard.press("Enter")
                    page.wait_for_selector(".andes-modal__content", state="hidden", timeout=10000)
                    print(f"✅ {rota_id} Salva!")
                    time.sleep(1) 
                except Exception as e:
                    print(f"⚠️ Erro na rota {rota_id}: {e}")
                    page.keyboard.press("Escape")

        # 🎯 PASSO 3: Mudar de Página
        print("🔍 Verificando se existe próxima página...")
        page.keyboard.press("End") # Garante que o botão Seguinte está visível
        time.sleep(1)
        
        btn_seguinte = page.get_by_text("Seguinte", exact=False)
        
        if btn_seguinte.is_visible() and btn_seguinte.is_enabled():
            print("➡️ Clicando em 'Seguinte'...")
            btn_seguinte.click()
            time.sleep(5) # Espera carregar a nova página
        else:
            print("🏁 Fim das páginas ou botão 'Seguinte' não encontrado.")
            break

    print("\n🏁 Operação concluída!")
    browser.close()
