[README.md](https://github.com/user-attachments/files/34991861/README.md)
# nibbit-vision# Nibbit Vision — inleverversie

Machine vision voor afzonderlijke Cheetos Nibbit sticks: segmentatie inclusief geschatte verborgen delen, zichtbaar/bedekt-status en lengte langs de middenlijn in mm op het horizontale vlak. Voor foto's en live kleurenbeeld. Deadline: 17 november 2026.

## Installeren en starten

1. Clone de meegeleverde Git-bundel met `git clone nibbit-vision.git.bundle nibbit-vision`, of pak de code-ZIP uit.
2. Pak de dataset-ZIP en weights-ZIP uit in deze projectmap. `data/`, `runs/`, `calibration/` en `nibbit/` moeten naast elkaar staan.
3. Installeer Python 3.12 (64 bit) op Windows. Voer `SETUP.cmd` uit en daarna `INSTALL_MODEL.cmd`. Internet is nodig voor de Python-pakketten. De laatste stap installeert PyTorch met CUDA 11.8 voor een geschikte NVIDIA-GPU. Een afzonderlijke CUDA-toolkit is niet nodig.
4. Sluit andere camera-apps en voer `START_GETRAIND_MODEL.cmd` uit. Kies de fysieke Lumina-camera (in de huidige opstelling index 0). Er opent een lokale browserinterface.

De exacte pakketversies staan in `requirements-base.lock.txt` en `requirements-model.lock.txt`. Op CPU kunnen de modelpakketten met `python -m pip install -r requirements-model.lock.txt` worden geïnstalleerd; detectie is dan aanzienlijk trager. Controleer de installatie met `.venv\Scripts\python.exe -m nibbit doctor`.

In de interface: **R** wist de verzamelde analyses, **S** bewaart een foto en **Q/Esc** sluit af. Hussel de sticks, haal je handen weg en druk op R. Houd de sticks tijdens het verzamelen van analyses stil. Herhaalde detecties worden gecombineerd; de modelgewichten veranderen niet. De weergegeven steun is geen kans op juistheid en een herhaalde fout kan blijven bestaan. Lengtes kunnen aanvullend worden geschat uit het masker of eerdere metingen; de interface toont hiervoor geen apart teken.

## Een opgeslagen foto verwerken

Voer vanuit de projectmap uit, met een foto uit dezelfde cameraopstelling:

```powershell
.venv\Scripts\python.exe -m nibbit infer --weights runs/bright-overlap-01/best.pt --image pad/naar/foto.png --setup-id lumina-training-bright-v1 --calibration calibration/lumina-training-bright-v1.json --threshold 0.3 --output runs/foto-resultaat
```

Dit bewaart ingetekende detecties, maskers en JSON-resultaten. Voor livebeeld laat je `--image` weg en gebruik je `--camera 0 --width 3840 --height 2160`. `--record` bewaart ook een livevideo en tijdsinformatie. `--device cpu` of `--device cuda` kiest expliciet de rekenhardware.

## Dataset en annotaties

Zie [docs/DATASET.md](docs/DATASET.md) voor formaat, herkomst, aantallen en splitsing. De definitieve trainingsinvoer is `data/bright-training-v1/annotations.json`: 149 beelden en 1573 instanties, waarvan 148 beelden voor training en één beeld met tien sticks voor validatie. Er zijn 100 samengestelde beelden en 49 echte beelden. De volledige dataset met bronbeelden wordt apart meegeleverd; behoud de relatieve paden.

```powershell
.venv\Scripts\python.exe -m nibbit validate --annotations data/bright-training-v1/annotations.json
```

Zichtbare delen en de volledige geschatte contour staan per stick in één instantie, ook als de zichtbare delen gescheiden zijn door een andere stick. Afgebroken losliggende stukken zijn afzonderlijke instanties. De verborgen contour in echte overlapbeelden is een schatting.

## Gebruikte gewichten en opnieuw trainen

Het definitieve model is `runs/bright-overlap-01/best.pt`. Architectuur: torchvision Mask R-CNN ResNet50-FPN, met achtergrond, volledig zichtbaar en deels bedekt als klassen; de maskers worden voor volledige, amodale segmentatie getraind. Segment Anything wordt niet voor detectie gebruikt. De oplossing gebruikt segmentatiemaskers en middenlijnen; geen oriented bounding boxes. Mask R-CNN gebruikt intern gewone voorstellen voor objectgebieden.

Het gewichtenarchief bevat ook `runs/pilot-synthetic-01/best.pt`, de initialisatie van de definitieve training, plus manifests, splits en trainingshistorie. Met de meegeleverde dataset en die initialisatie:

```powershell
.venv\Scripts\python.exe -m nibbit train --annotations data/bright-training-v1/annotations.json --initial-weights runs/pilot-synthetic-01/best.pt --output runs/reproduced-bright --epochs 8 --min-size 720 --max-size 1280 --lr 0.00003 --accumulate 2 --seed 42 --device cuda
```

Deze training gebruikt AdamW, batchgrootte 1 en gradiëntaccumulatie over twee beelden. De beste epoch wordt gekozen op validatie-F1, vervolgens statusnauwkeurigheid en masker-IoU. `split.json`, `history.json` en `model_manifest.json` leggen de oorspronkelijke run vast. De seed ligt vast; GPU-berekeningen kunnen tussen systemen toch verschillen. Exact dezelfde binaire gewichten na hertraining zijn niet gegarandeerd.

De eerste training kan afzonderlijk opnieuw worden uitgevoerd met:

```powershell
.venv\Scripts\python.exe -m nibbit train --annotations data/synthetic-pilot-v1/annotations.json --output runs/reproduced-pilot --epochs 5 --min-size 384 --max-size 640 --seed 42 --device cuda
```

Zonder `--initial-weights` wordt een pretrained COCO-model gedownload. Voor het herhalen van de definitieve training vanuit de meegeleverde proefgewichten is deze download niet nodig.

## Lengtemeting en kalibratie

De middenlijn wordt naar het gekalibreerde horizontale vlak omgerekend en langs de lijn opgeteld. Fragmenten korter dan 5 mm mogen worden genegeerd. Verborgen delen worden geschat. Als de primaire middenlijnmeting faalt, probeert de code een aanvullende schatting uit het masker; er wordt geen willekeurige standaardlengte gebruikt. Zonder geldige kalibratie kan geen betekenisvolle mm-lengte worden geleverd.

De huidige kalibratie is voor 3840×2160, setup `lumina-training-bright-v1`, met markeringen op 400×225 mm. De gebruiker heeft de schaal praktisch gecontroleerd. De opgegeven referentietolerantie was ongeveer 5 mm. Camerapositie, zoom en uitsnede moeten gelijk blijven. De kalibratie corrigeert perspectief in het vlak, maar niet lensvervorming en hoogteverschillen van gestapelde sticks.

Na verplaatsen: maak een nieuwe markerfoto en kalibreer met werkelijk gemeten afstanden tussen de markercentra:

```powershell
.venv\Scripts\python.exe -m nibbit calibrate --image nieuwe-markerfoto.png --width-mm 400 --height-mm 225 --setup-id nieuwe-opstelling --output calibration/nieuwe-opstelling.json
```

Klik de vier referentiepunten volgens de aanwijzingen. Controleer met een bekende lengte in beide richtingen en gebruik vervolgens deze nieuwe kalibratie en setup-id bij detectie.

## Controle en beperkingen

```powershell
.venv\Scripts\python.exe -m unittest discover -s tests -v
```

De oorspronkelijke modelmanifest rapporteert op **slechts één validatiebeeld met tien sticks**: 10 juiste detecties, 0 extra, 0 gemiste, gemiddelde gematchte amodale IoU 0.903 en correcte status bij alle tien matches, bij scoregrens 0.5. Dit bewijst geen foutloze werking op willekeurige hopen met twintig nieuwe sticks. De livegrens is 0.3 en de huidige code bevat extra maskerdeduplicatie en temporele combinatie; bovenstaande historische score is geen nieuwe meting van die volledige liveketen.

Eerdere challengebeelden zijn voor foutanalyse gebruikt en zijn geen onafhankelijke eindtest meer. Dubbeltellingen, gemiste objecten, verkeerde zichtbaarheidsstatus en onjuiste reconstructie blijven mogelijk, vooral bij bijna evenwijdige aanraking, korte stukken en sterke overlap. Controleer aantallen, status en lengtes met nieuwe schikkingen en nieuwe fysieke sticks. Een correct totaal alleen garandeert geen correcte individuele segmentatie.

## Bestanden en inleveren

- `nibbit/`: camera, model, maskers, lengte, temporele combinatie en interface.
- `scripts/`: datasetvoorbereiding, review en verpakking.
- `tests/`: automatische controles.
- `calibration/`: opstellingsgegevens en referenties.
- `review/`: ontwikkelcontroles.
- `docs/DEVELOPMENT_HISTORY.md`: oude ontwikkelnotities; historische fouten en resultaten zijn geen actuele gebruikersinstructies.

Data en gewichten staan buiten Git om de repository hanteerbaar te houden. De drie archieven samen bevatten de reproduceerbare inzending. De Git-bundel bevat de echte lokale repository; er is nog geen externe Git-hosting ingesteld. Na eventuele wijzigingen: commit de code en bouw opnieuw met `python scripts/package_submission.py --output ../inlevering-nibbit-vision`. Het pakket bevat controlesommen en de commit-id.

Voor de fysieke demo blijven een vaste kleuren-cameraopstelling en een zak Nibbit sticks nodig. Controleer daarnaast de apart gevraagde resultaatfoto's en livevideo voor de uiteindelijke inzending.
