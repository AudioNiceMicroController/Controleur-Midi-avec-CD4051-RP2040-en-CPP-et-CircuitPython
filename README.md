# CD4051 avec RP2040 (raspberry pico)

## Installer circuitpython
 - Il faut aller sur https://circuitpython.org/board/raspberry_pi_pico/ et télécharger la dernière version. Débrancher le pico, appuyer sur le bouton en le rebranchant (mode DFU). Glisser déposer le .uf2.
 - Télécharger https://github.com/adafruit/Adafruit_CircuitPython_Bundle/releases. Copier adafruit_midi dans lib qui est à la racide du RP2040.

## Thonny IDE
[Télécharger ici](https://thonny.org). Lors de la programmation il faudra bien choisir circuitpython et non pas micropython.

## Branchements du/des CD4051
![image.png](./img.jpg)

## Programme pour un potentiomètre diectement branché
```
# Version Control Change (CC)
import board
import analogio
import time
import usb_midi
import adafruit_midi
from adafruit_midi.control_change import ControlChange

# Initialisation MIDI
midi = adafruit_midi.MIDI(midi_out=usb_midi.ports[1], out_channel=0)
potentiometre = analogio.AnalogIn(board.A3)

# Configuration
CONTROLEUR_MIDI = 1  # CC1 = Modulation Wheel
valeur_precedente = -1
seuil = 1000  # Seuil pour éviter l'envoi trop fréquent

print("Contrôleur MIDI CC - Potentiomètre")
print("----------------------------------")

try:
    while True:
        valeur = potentiometre.value
        valeur_cc = int((valeur / 65535) * 127)
        
        # Envoyer seulement si changement significatif
        if abs(valeur - valeur_precedente) > seuil:
            midi.send(ControlChange(CONTROLEUR_MIDI, valeur_cc))
            print(f"CC{CONTROLEUR_MIDI}: {valeur_cc:3d}")
            valeur_precedente = valeur
        
        time.sleep(0.02)
        
except KeyboardInterrupt:
    print("\nArrêt du contrôleur CC.")
```

## Programme pour un potentiomètre diectement branché sur CD4051
```
# Contrôleur MIDI 8 canaux avec CD4051
# Envoie des messages Control Change en temps réel

import board
import analogio
import digitalio
import time
import usb_midi
import adafruit_midi
from adafruit_midi.control_change import ControlChange

# Configuration MIDI
CC_BASE = 20  # Contrôleurs CC20 à CC27
SEUIL = 1000  # Seuil de changement pour éviter le bruit
CANAL_MIDI = 3  # Canal MIDI 4

# Configuration MIDI
midi = adafruit_midi.MIDI(midi_out=usb_midi.ports[1], out_channel=CANAL_MIDI)

# Configuration des broches du CD4051
S0 = digitalio.DigitalInOut(board.GP0)  # Broche A0 (LSB)
S1 = digitalio.DigitalInOut(board.GP1)  # Broche A1
S2 = digitalio.DigitalInOut(board.GP2)  # Broche A2 (MSB)
INH = digitalio.DigitalInOut(board.GP3)  # Broche INH (Inhibit)

# Configuration des broches en sortie
for pin in [S0, S1, S2, INH]:
    pin.direction = digitalio.Direction.OUTPUT

# Entrée analogique
adc = analogio.AnalogIn(board.A0)



def activer_multiplexeur(actif):
    """Active ou désactive le multiplexeur via INH"""
    INH.value = not actif  # LOW = actif, HIGH = désactivé
    time.sleep(0.0001)     # Délai de stabilisation

def selectionner_canal(canal):
    """Sélectionne un canal spécifique (0-7) avec méthode bit à bit"""
    # Désactiver pendant le changement
    activer_multiplexeur(False)
    
    # 🎯 MÉTHODE BIT À BIT EXCELLENTE !
    S0.value = (canal >> 0) & 1  # Bit 0 (LSB)
    S1.value = (canal >> 1) & 1  # Bit 1
    S2.value = (canal >> 2) & 1  # Bit 2 (MSB)
    
    # Réactiver après changement
    activer_multiplexeur(True)
    time.sleep(0.0002)  # Délai de stabilisation

def valeur_vers_midi(valeur_adc):
    """Convertit la valeur ADC (0-65535) en valeur MIDI (0-127)"""
    return max(0, min(127, int((valeur_adc / 65535) * 127)))

# Initialisation
activer_multiplexeur(False)
canal_actuel = 0
valeur_prec = [0] * 8  # 8 valeurs précédentes
derniere_valeur_midi = [-1] * 8  # Dernières valeurs MIDI envoyées

print("Contrôleur MIDI CD4051 - CC20 à CC27")
print("====================================")
print("Envoi des messages Control Change en temps réel")
print("Seuil de sensibilité:", SEUIL)
print()

ports = [5]

try:
    while True:
        for e in ports:
            canal_actuel = e
            # Sélectionner et lire le canal actuel
            selectionner_canal(canal_actuel)
            valeur_adc = adc.value
            
            # Vérifier le changement significatif
            if abs(valeur_adc - valeur_prec[canal_actuel]) > SEUIL:
                valeur_midi = valeur_vers_midi(valeur_adc)
                cc_num = CC_BASE + canal_actuel
                midi.send(ControlChange(cc_num, valeur_midi))
                
                # Mettre à jour les valeurs
                valeur_prec[canal_actuel] = valeur_adc
                derniere_valeur_midi[canal_actuel] = valeur_midi
                
                # Affichage console
                pourcentage = (valeur_midi / 127) * 100
                print(f"Canal {canal_actuel} → CC{cc_num}: {valeur_midi:3d}/127 ({pourcentage:5.1f}%)")
        
            time.sleep(0.001)  # Court délai entre les canaux
        
except KeyboardInterrupt:
    # Arrêt propre - désactiver le multiplexeur
    activer_multiplexeur(False)
    
    # Envoyer toutes les valeurs à 0
    for canal in range(8):
        midi.send(ControlChange(CC_BASE + canal, 0, channel=CANAL_MIDI))
    
    print("\n📴 Multiplexeur désactivé")
    print("🎵 Tous les contrôleurs MIDI remis à zéro")
    print("Arrêt du programme.")
```