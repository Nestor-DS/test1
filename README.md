```

import random
import time
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.common.exceptions import TimeoutException, StaleElementReferenceException
from webdriver_manager.chrome import ChromeDriverManager
from selenium.webdriver.chrome.service import Service

# ---------- CONFIGURACIÓN ----------
PIN = "123456"          # Cambia por el PIN de la partida
USERNAME = "Bot"        # Cambia por el nombre que quieras
DRIVER_PATH = None      # Si usas webdriver-manager, déjalo en None
# -----------------------------------

class KahootBot:
    def __init__(self):
        # Configura el driver (usa ChromeDriverManager para que sea automático)
        service = Service(ChromeDriverManager().install())
        self.driver = webdriver.Chrome(service=service)
        self.wait = WebDriverWait(self.driver, 10)
        self.driver.maximize_window()

    def join_game(self, pin, username):
        """Abre Kahoot, introduce el PIN y el nombre de usuario."""
        self.driver.get("https://kahoot.it/")
        
        # 1. Ingresar PIN
        try:
            pin_input = self.wait.until(
                EC.visibility_of_element_located((By.CSS_SELECTOR, "input[data-functional-selector='join-game-input']"))
            )
            pin_input.send_keys(pin)
            
            join_btn = self.driver.find_element(By.CSS_SELECTOR, "button[data-functional-selector='join-game-button']")
            join_btn.click()
        except TimeoutException:
            print("❌ No se pudo cargar el campo del PIN. Revisa la conexión.")
            self.driver.quit()
            return False

        # 2. Ingresar nombre de usuario
        try:
            username_input = self.wait.until(
                EC.visibility_of_element_located((By.CSS_SELECTOR, "input[data-functional-selector='join-game-username-input']"))
            )
            username_input.send_keys(username)
            
            submit_btn = self.driver.find_element(By.CSS_SELECTOR, "button[data-functional-selector='join-game-username-submit']")
            submit_btn.click()
        except TimeoutException:
            print("❌ No se pudo cargar el campo del nombre. Puede que el PIN sea incorrecto.")
            self.driver.quit()
            return False

        print(f"✅ Unido a la partida como '{username}'. Esperando a que comience...")
        return True

    def wait_for_next_question(self):
        """
        Espera hasta que los botones de respuesta estén disponibles.
        Retorna una lista de botones si los encuentra, o None si el juego terminó.
        """
        try:
            # Esperamos a que aparezca al menos un botón de respuesta (answer-0)
            self.wait.until(
                EC.presence_of_element_located((By.CSS_SELECTOR, "button[data-functional-selector='answer-0']"))
            )
            # Recogemos todos los botones de respuesta disponibles (pueden ser 2, 3 o 4)
            answer_buttons = self.driver.find_elements(By.CSS_SELECTOR, "button[data-functional-selector^='answer-']")
            # Filtramos solo los que están visibles y habilitados
            available = [btn for btn in answer_buttons if btn.is_displayed() and btn.is_enabled()]
            return available
        except TimeoutException:
            # Si no aparecen botones, puede que el juego haya terminado (pódium o game over)
            return None

    def is_game_over(self):
        """Detecta si la partida ha terminado (pantalla de pódium o game over)."""
        try:
            # Busca elementos típicos del final del juego
            self.driver.find_element(By.CSS_SELECTOR, "div[data-functional-selector='game-over-screen'], div[data-functional-selector='podium-screen']")
            return True
        except:
            return False

    def play(self):
        """Bucle principal: responde preguntas hasta que termina el juego."""
        round_num = 0
        while True:
            # Comprobamos si ya terminó antes de esperar siguiente pregunta
            if self.is_game_over():
                print("🏁 Partida finalizada. ¡Hasta la próxima!")
                break

            print(f"⏳ Esperando siguiente pregunta...")
            buttons = self.wait_for_next_question()

            if buttons is None:
                # Si no hay botones, puede que esté en transición o haya terminado
                if self.is_game_over():
                    print("🏁 Partida finalizada.")
                else:
                    print("⏳ Esperando a que los demás terminen...")
                    time.sleep(2)
                continue

            # Si hay botones, elegimos uno al azar
            if len(buttons) > 0:
                chosen = random.choice(buttons)
                try:
                    # Hacemos scroll y click
                    self.driver.execute_script("arguments[0].scrollIntoView(true);", chosen)
                    chosen.click()
                    round_num += 1
                    print(f"🎯 Ronda {round_num}: Respuesta seleccionada (opción {buttons.index(chosen) + 1})")
                except StaleElementReferenceException:
                    # Si el elemento se actualiza, reintentamos en la siguiente iteración
                    print("🔄 Elemento obsoleto, reintentando...")
                    continue

            # Esperamos un momento para que se registre la respuesta y pase a la siguiente
            time.sleep(1.5)

    def run(self):
        """Ejecuta todo el flujo."""
        try:
            if self.join_game(PIN, USERNAME):
                self.play()
        except Exception as e:
            print(f"⚠️ Error inesperado: {e}")
        finally:
            # Esperamos 3 segundos para ver el resultado final y cerramos
            time.sleep(3)
            self.driver.quit()
            print("🔚 Navegador cerrado.")

if __name__ == "__main__":
    bot = KahootBot()
    bot.run()

```