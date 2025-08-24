substitutions:
  name: esphome-web-a86efc
  friendly_name: AutoStopStart

esphome:
  name: ${name}
  friendly_name: ${friendly_name}
  min_version: 2024.6.0
  name_add_mac_suffix: false
  project:
    name: esphome.web
    version: dev
  on_boot:
    priority: -100  # Exécuté très tôt pour garantir que les sliders reflètent les valeurs globales
    then:
      - number.set:
          id: delay_on_slider
          value: !lambda 'return id(delay_on);'
      - number.set:
          id: delay_off_slider
          value: !lambda 'return id(delay_off);'
      - number.set:
          id: distance_slider
          value: !lambda 'return id(distance);'
      - output.set_level:
          id: lcd_backlight
          level: 0.5   # Luminosité initiale de l'écran LCD

esp32:
  board: esp32dev
  framework:
    type: esp-idf
    version: recommended

# Logs série
logger:

# Intégration Home Assistant
api:

# Mise à jour OTA
ota:
  - platform: esphome

# Provisionnement Wi-Fi via port série
improv_serial:

# Point d'accès Wi-Fi pour configuration
wifi:
  ap: {}

captive_portal:   # Portail captif pour provisionner le Wi-Fi

# Import Dashboard (utile pour mise à jour depuis l’UI ESPHome)
dashboard_import:
  package_import_url: github://esphome/example-configs/esphome-web/esp32.yaml@main
  import_full_config: true

# Webserver local pour debug
web_server:

# === Variables globales ===
globals:
  - id: countdown_timer
    type: int
    restore_value: yes
    initial_value: '0'
  - id: relay_on
    type: bool
    restore_value: yes
    initial_value: '0'
  - id: delay_on        # Temps avant allumage relais
    type: int
    restore_value: yes
    initial_value: '120'
  - id: delay_off       # Temps avant extinction relais
    type: int
    restore_value: yes
    initial_value: '20'
  - id: distance        # Distance de déclenchement en cm
    type: int
    restore_value: yes
    initial_value: '300'
  - id: emergency_active
    type: bool
    restore_value: no
    initial_value: 'false'

# === Bus I2C pour l’écran LCD ===
i2c:
  sda: GPIO21
  scl: GPIO22
  scan: true

# === Afficheur LCD ===
display:
  - platform: lcd_pcf8574
    id: lcd_display
    dimensions: 16x2
    address: 0x27
    update_interval: 1s
    lambda: |-
      // 🔴 Si arrêt d'urgence actif : priorité absolue
      if (id(emergency_active)) {
        it.print(12, 0, "STOP");    // Affiche STOP
        it.print(0, 1, "OFF");      // Forcer affichage OFF
        it.printf(4, 1, "[0] %3ds", id(countdown_timer)); 
        return; // ⛔ Sortie immédiate : aucune autre logique exécutée
      }

      // ⏱️ Décrément du compteur uniquement si pas en urgence
      if (id(countdown_timer) > 0) {
        id(countdown_timer) -= 1;
      }

      // 📏 Affichage distance
      if (id(moving_distance).has_state()) {
        it.printf(0, 0, "%3.0fcm", id(moving_distance).state);

        // ⚡ Allumage relais si distance < seuil et timer expiré
        if (id(moving_distance).state < id(distance) && id(countdown_timer) == 0) {
          id(relay).turn_on();
          id(countdown_timer) = id(delay_on);
        }
      }

      // ⏹️ Extinction relais si timer écoulé
      if (id(relay).state && id(countdown_timer) == 0) {
        id(relay).turn_off();
        id(countdown_timer) = id(delay_off);
      }

      // 💡 Affichage état relais et compteur
      it.print(0, 1, id(relay).state ? "ON " : "OFF");
      it.printf(4, 1, "[%1d] %3ds", id(relay).state ? 1 : 0, id(countdown_timer));

# === Rétroéclairage LCD ===
output:
  - platform: ledc
    id: lcd_backlight
    pin: GPIO32
    frequency: 1000Hz

light:
  - platform: monochromatic
    output: lcd_backlight
    name: "LCD Backlight"

# === Capteur LD2410 (présence) ===
uart:
  tx_pin: GPIO18
  rx_pin: GPIO19
  baud_rate: 256000  # Communication rapide pour LD2410

ld2410:

# === Relais de commande ===
switch:
  - platform: gpio
    pin: GPIO13
    name: "Relay"
    id: relay
    # Sécurité : extinction auto après 5 minutes
    on_turn_on:
      - delay: 300000ms
      - switch.turn_off: relay

  - platform: ld2410
    engineering_mode:
      name: "Engineering Mode"
    bluetooth:
      name: "LD2410 Bluetooth"

# === Capteurs LD2410 ===
sensor:
  - platform: ld2410
    light:
      name: Ambient Light
    moving_distance:
      id: moving_distance
      name : "Moving Distance"
    still_distance:
      name: "Still Distance"
    moving_energy:
      name: "Moving Energy"
    still_energy:
      name: "Still Energy"
    detection_distance:
      id: ld2410_distance
      name: "Detection Distance"

    # Énergie par zones g0 → g8
    g0: { move_energy: { name: "g0 Move Energy" }, still_energy: { name: "g0 Still Energy" } }
    g1: { move_energy: { name: "g1 Move Energy" }, still_energy: { name: "g1 Still Energy" } }
    g2: { move_energy: { name: "g2 Move Energy" }, still_energy: { name: "g2 Still Energy" } }
    g3: { move_energy: { name: "g3 Move Energy" }, still_energy: { name: "g3 Still Energy" } }
    g4: { move_energy: { name: "g4 Move Energy" }, still_energy: { name: "g4 Still Energy" } }
    g5: { move_energy: { name: "g5 Move Energy" }, still_energy: { name: "g5 Still Energy" } }
    g6: { move_energy: { name: "g6 Move Energy" }, still_energy: { name: "g6 Still Energy" } }
    g7: { move_energy: { name: "g7 Move Energy" }, still_energy: { name: "g7 Still Energy" } }
    g8: { move_energy: { name: "g8 Move Energy" }, still_energy: { name: "g8 Still Energy" } }

# === Détection de présence ===
binary_sensor:
  - platform: ld2410
    has_target:
      id: Presence
      name: "Presence"
    has_moving_target:
      id: Move
      name: "Moving Target"
    has_still_target:
      id: still
      name: "Still Target"
    out_pin_presence_status:
      name: "Out Pin Presence Status"

# === Arrêt d'urgence ===
  - platform: gpio
    pin:
      number: GPIO23
      mode: INPUT_PULLUP
      inverted: true
    id: emergency_stop
    name: "Emergency Stop Button"

    on_press:
      then:
        - logger.log: "⚠️ ARRET D'URGENCE ACTIVÉ !"
        - switch.turn_off: relay
        - globals.set:
            id: relay_on
            value: 'false'
        - globals.set:
            id: countdown_timer
            value: !lambda 'return id(delay_off);'
        - lambda: |-
            // Forcer l'affichage immédiat STOP
            id(lcd_display).print(12, 0, "STOP");

    on_release:
      then:
        - logger.log: "✅ Bouton d'urgence relâché"
        - lambda: |-
            // Effacer STOP immédiatement
            id(lcd_display).print(12, 0, "    ");

# === Infos système ===
text_sensor:
  - platform: wifi_info
    ip_address:
      name: "${friendly_name} IP Address"
      id: device_ip_address

# === Paramètres configurables depuis HA ===
number:
  - platform: template
    name: "Delay ON (s)"
    id: delay_on_slider
    min_value: 1
    max_value: 600
    step: 1
    initial_value: 120
    restore_value: true
    optimistic: true
    on_value:
      - globals.set:
          id: delay_on
          value: !lambda 'return (int)x;'

  - platform: template
    name: "Delay OFF (s)"
    id: delay_off_slider
    min_value: 1
    max_value: 600
    step: 1
    initial_value: 20
    restore_value: true
    optimistic: true
    on_value:
      - globals.set:
          id: delay_off
          value: !lambda 'return (int)x;'

  - platform: template
    name: "Distance (cm)"
    id: distance_slider
    min_value: 1
    max_value: 600
    step: 1
    initial_value: 150
    restore_value: true
    optimistic: true
    on_value:
      - globals.set:
          id: distance
          value: !lambda 'return (int)x;'

# === Bouton redémarrage ===
button:
  - platform: restart
    name: "Restart Device"
