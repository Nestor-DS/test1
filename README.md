```
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.common.keys import Keys
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

PIN = "123456"          # PIN de prueba
NICKNAME = "BotDemo"

driver = webdriver.Chrome()
wait = WebDriverWait(driver, 20)

try:
    driver.get("https://kahoot.it")

    # Introducir PIN
    pin_input = wait.until(
        EC.presence_of_element_located((By.TAG_NAME, "input"))
    )
    pin_input.send_keys(PIN)
    pin_input.send_keys(Keys.ENTER)

    # Introducir nickname
    nickname_input = wait.until(
        EC.presence_of_element_located((By.TAG_NAME, "input"))
    )
    nickname_input.send_keys(NICKNAME)
    nickname_input.send_keys(Keys.ENTER)

    print("Sesión de prueba iniciada correctamente.")

    # Mantener el navegador abierto para inspección
    input("Pulsa Enter para cerrar...")

finally:
    driver.quit()

```