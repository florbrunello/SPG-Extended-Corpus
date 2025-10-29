# SPG-Extended-Corpus

An extended text corpus with annotations in the BRAT standoff format. Each annotated document consists of a plain-text file and a corresponding `.ann` file sharing the same basename.

If you use this corpus in research, please see Citation below and open an issue/PR to add the exact reference once available.

## Repository structure

- `txt/` — Raw text files (`.txt`). One file per document.
- `brat/` — BRAT annotation files (`.ann`). One file per document, same basename as its corresponding `.txt`.
- `README.md` — This file.

Example pairing:

- `txt/299966440.txt` ↔ `brat/299966440.ann`

> Note: Ensure that for every `.ann` there is a matching `.txt` with the same filename stem, and vice versa.

## BRAT .ann quick reference

BRAT annotations are stored in standoff format in `.ann` files:

- Text-bound annotations (entities):
  - `T1	Label start end	mention text`
- Relations between annotations:
  - `R1	Relation Arg1:T1 Arg2:T2`
- Events (triggered by a text-bound node):
  - `E1	Trigger:T3 Role1:T4 Role2:T5`
- Attributes (properties on T/R/E):
  - `A1	Attribute TargetID Value`
- Normalizations / external links:
  - `N1	Reference T1 Wikipedia:12345`  

Minimal illustrative example:

```
T1	Person 0 4	John
T2	Organization 13 19	AcmeCo
R1	Works_For Arg1:T1 Arg2:T2
A1	Negation R1 True
```

For the full BRAT specification and UI, see: https://brat.nlplab.org/standoff.html

## How to view/edit the annotations

1. Install the BRAT rapid annotation tool (web-based) by following the official setup guide: https://brat.nlplab.org/installation.html
2. Place the `txt/` and `brat/` contents into a BRAT project directory (or symlink them).
3. Start the BRAT server and open the project in your browser to visualize and edit annotations.

## Programmatic access (Python)

Below is a minimal example to iterate over document pairs and parse basic entity lines from `.ann` files. Adjust to your schema.

```python
from pathlib import Path

TXT_DIR = Path("txt")
ANN_DIR = Path("brat")

for ann_path in sorted(ANN_DIR.glob("*.ann")):
    stem = ann_path.stem
    txt_path = TXT_DIR / f"{stem}.txt"
    if not txt_path.exists():
        print(f"Warning: missing text for {ann_path.name}")
        continue

    text = txt_path.read_text(encoding="utf-8", errors="replace")

    entities = []
    with ann_path.open(encoding="utf-8", errors="replace") as f:
        for line in f:
            if line.startswith("T"):
                # Example: T1\tLabel start end\tmention
                try:
                    tid, rest = line.strip().split("\t", 1)
                    span, mention = rest.split("\t", 1)
                    label, start, end = span.split()
                    entities.append({
                        "id": tid,
                        "label": label,
                        "start": int(start),
                        "end": int(end),
                        "text": mention,
                    })
                except ValueError:
                    pass  # handle multi-span or other cases as needed

    # Do something with text and entities
    print(stem, len(entities))
```

## Dataset creation and provenance

- Source generator: SPG-Synthetic-Patient-Generator (https://github.com/florbrunello/SPG-Synthetic-Patient-Generator)
- Model for text extension: Llama 3.1 8B (Meta)
- Volume: 1,000 documents
- Language: Spanish

Pipeline overview:
1. Generate initial synthetic discharge reports with SPG, producing document IDs and base annotations.
2. Produce plain-text reports in `txt/` for each document ID.
3. Extend and enrich the “Informe clínico del paciente” section with Llama 3.1 8B using the prompt in `prompts/text_extension_prompt_es.txt`.
4. Generate an intermediate inline-XML version using the MEDDOCAN tagging prompt in `prompts/xml_tagging_prompt_es.txt`.
5. Convert the inline XML to BRAT standoff annotations and save them as `.ann` in `brat/` with the same basename as the corresponding `.txt`.
6. Quality control and cleanup: fix formatting, remove malformed outputs, and delete any files that contained refusal strings such as "No puedo cumplir esta tarea".

Files per document:
- `txt/<id>.txt` — the full clinical report in plain text.
- `brat/<id>.ann` — BRAT standoff annotations converted from the intermediate inline XML (MEDDOCAN tags).

### Prompts used (Spanish)

- Text extension prompt: see `prompts/text_extension_prompt_es.txt`
- XML tagging prompt (MEDDOCAN): see `prompts/xml_tagging_prompt_es.txt`

Text extension prompt (used to produce the final structured report content in `txt/`):

```text
Estoy generando un dataset sintético de informes clínicos de alta médica después de una internación, comparable con el dataset MEDDOCAN, para evaluar sistemas de procesamiento de texto clínico con datos no vistos. 
Todos los informes tienen que tener las siguientes secciones: Datos del paciente (con nombre, dni, fecha de nacimiento, género, domicilio, ciudad, código postal, email, teléfono fijo, teléfono móvil, número de historia clínica (NHC), NASS y condición de riesgo), Datos asistenciales (con información sobre el médico, incluyendo nombre, NC y su cargo y dirección), Fecha de ingreso, Centro de salud de ingreso e Informe clínico del paciente. Opcionalmente, también puede tener las secciones "Antecedentes", "Evolución y comentarios" y "Remitido por".        
Me interesa que la sección "Informe clínico del paciente" cumpla con estos requisitos:
- entre 100 y 1200 palabras, con una longitud promedio de 375 palabras.
- incluir información detallada sobre la condición médica del paciente, antecedentes relevantes, evolución clínica, exploración física, pruebas diagnósticas, tratamiento y pronóstico.
- redacción clara, técnica y coherente con la terminología médica.
- variabilidad en la estructura, el nivel de detalle y la redacción para evitar informes repetitivos o mecánicos.
- no incluyas ningún tipo de introducción, explicación o enunciado fuera del formato del informe. 
- Todos los informes deben respetar estrictamente el formato dado en el ejemplo sin agregar introducciones, comentarios meta, encabezados explicativos ni frases como "Aquí te presento un ejemplo". 
- Los informes deben comenzar directamente con las secciones estructuradas y sus contenidos.

Ejemplo de informe original:

Datos del paciente.
Nombre: María Soledad Moreno Roca
DNI: 23556552K
Fecha de nacimiento: 09/01/1941
Género: Mujer
Domicilio: Calle de Almagro 80
Ciudad: Denia, Valencia, Comunidad Valenciana
Código postal: 46571
Email: mariasoledad_roca@ucm.es
Teléfono fijo: +34 960 66 89 48
Teléfono móvil: +34 660 57 14 97
NHC: 2409425
NASS: 468043486571
Condición de riesgo: Científico de Investigación

Datos asistenciales.
Médico: Dr. Juan Ramón Benito Vicente. NC 097900390. Investigador Clínico en Epidemiología. Instituto de Investigación Biomédica en Red de Enfermedades Infecciosas (CIBERINFEC). Avenida Monforte de Lemos 3-5. 28029. Madrid. España.
Fecha de ingreso: 05/06/1996
Centro de salud: Centro de Salud Carabanchel
Informe clínico del paciente:
Paciente sobreviviente de violencia de 55 años de edad, acompañado de su madre. 

Ejemplo de informe extendido:

Datos del paciente.
Nombre:  Jose .
Apellidos: Aranda Martinez.
NHC: 2748903.
NASS: 26 37482910 04.
Domicilio: Calle Losada Martí 23, 5 B..
Localidad/ Provincia: Madrid.
CP: 28016.

Datos asistenciales.
Fecha de nacimiento: 15/04/1977.
País: España.
Edad: 37 años Sexo: F.
Fecha de Ingreso: 05/06/2018.
Médico: María Merino Viveros  NºCol: 28 28 35489.
Informe clínico del paciente: varón de 37 años con vida previa activa que refiere dolores osteoarticulares de localización variable en el último mes y fiebre en la última semana con picos (matutino y vespertino) de 40 C las últimas 24-48 horas, por lo que acude al Servicio de Urgencias. Antes de comenzar el cuadro estuvo en Extremadura en una región endémica de brucella, ingiriendo leche de cabra sin pasteurizar y queso de dicho ganado. Entre los comensales aparecieron varios casos de brucelosis. Durante el ingreso para estudio del síndrome febril con antecedentes epidemiológicos de posible exposición a Brucella presenta un cuadro de orquiepididimitis derecha.
La exploración física revela: Tª 40,2 C; T.A: 109/68 mmHg; Fc: 105 lpm. Se encuentra consciente, orientado, sudoroso, eupneico, con buen estado de nutrición e hidratación. En cabeza y cuello no se palpan adenopatías, ni bocio ni ingurgitación de vena yugular, con pulsos carotídeos simétricos. Auscultación cardíaca rítmica, sin soplos, roces ni extratonos. Auscultación pulmonar con conservación del murmullo vesicular. Abdomen blando, depresible, sin masas ni megalias. En la exploración neurológica no se detectan signos meníngeos ni datos de focalidad. Extremidades sin varices ni edemas. Pulsos periféricos presentes y simétricos. En la exploración urológica se aprecia el teste derecho aumentado de tamaño, no adherido a piel, con zonas de fluctuación e intensamente doloroso a la palpación, con pérdida del límite epidídimo-testicular y transiluminación positiva.
Los datos analíticos muestran los siguentes resultados: Hemograma: Hb 13,7 g/dl; leucocitos 14.610/mm3 (neutrófilos 77%); plaquetas 206.000/ mm3. VSG: 40 mm 1ª hora. Coagulación: TQ 87%; TTPA 25,8 seg. Bioquímica: Glucosa 117 mg/dl; urea 29 mg/dl; creatinina 0,9 mg/dl; sodio 136 mEq/l; potasio 3,6 mEq/l; GOT 11 U/l; GPT 24 U/l; GGT 34 U/l; fosfatasa alcalina 136 U/l; calcio 8,3 mg/dl. Orina: sedimento normal.
Durante el ingreso se solicitan Hemocultivos: positivo para Brucella y Serologías específicas para Brucella: Rosa de Bengala +++; Test de Coombs > 1/1280; Brucellacapt > 1/5120. Las pruebas de imagen solicitadas ( Rx tórax, Ecografía abdominal, TAC craneal, Ecocardiograma transtorácico) no evidencian patología significativa, excepto la Ecografía testicular, que muestra engrosamiento de la bolsa escrotal con pequeña cantidad de líquido con septos y testículo aumentado de tamaño con pequeñas zonas hipoecoicas en su interior que pueden representar microabscesos.
Con el diagnóstico de orquiepididimitis secundaria a Brucella se instaura tratamiento sintomático (antitérmicos, antiinflamatorios, reposo y elevación testicular) así como tratamiento antibiótico específico: Doxiciclina 100 mg vía oral cada 12 horas (durante 6 semanas) y Estreptomicina 1 gramo intramuscular cada 24 horas (durante 3 semanas). El paciente mejora significativamente de su cuadro tras una semana de ingreso, decidiéndose el alta a su domicilio donde completó la pauta de tratamiento antibiótico. En revisiones sucesivas en consultas se constató la completa remisión del cuadro.
```

XML tagging prompt (used to produce the intermediate inline XML with MEDDOCAN tags, later converted into `.ann`):

```text
Quiero que identifiques entidades nombradas que requieren ser anonimizadas en el informe clínico que copio entre comillas al final de esta instrucción. Quiero que me des el resultado en formato .xml in-line, donde las entidades sean identificadas por etiquetas en el mismo texto. Quiero que etiquetes con los criterios MEDDOCAN. A continuación, te muestro un ejemplo que contiene:
- El texto original del informe en formato plano (.txt)
- La representación estructurada del mismo en XML con etiquetas semánticas detalladas y posiciones de texto (atributos start, end, text, TYPE, etc.).
Tu tarea será generar un XML con las mismas reglas de estructura y etiquetado a partir de cada texto clínico. Instrucciones:
- Conserva el formato exacto del XML del ejemplo.
- Cada etiqueta tiene que tener el tipo de entidad (TYPE) del inventario de MEDDOCAN. Los tipos de entidad que puedes usar son los siguientes: 
NOMBRE_SUJETO_ASISTENCIA
EDAD_SUJETO_ASISTENCIA
SEXO_SUJETO_ASISTENCIA
FAMILIARES_SUJETO_ASISTENCIA
NOMBRE_PERSONAL_SANITARIO
FECHAS
PROFESION
HOSPITAL
CENTRO_SALUD
INSTITUCION
CALLE
TERRITORIO
PAIS
NUMERO_TELEFONO
NUMERO_FAX
CORREO_ELECTRONICO
ID_SUJETO_ASISTENCIA
ID_CONTACTO_ASISTENCIAL
ID_ASEGURAMIENTO
ID_TITULACION_PERSONAL_SANITARIO
ID_EMPLEO_PERSONAL_SANITARIO
IDENTIF_VEHICULOS_NRSERIE_PLACAS
IDENTIF_DISPOSITIVOS_NRSERIE
DIREC_PROT_INTERNET
URL_WEB
IDENTIF_BIOMETRICOS
OTRO_NUMERO_IDENTIF
OTROS_SUJETO_ASISTENCIA
  - y un campo de comentario (comment) vacío
Cuando te dé un nuevo texto, responde solo con el XML, sin explicaciones adicionales.

Ejemplo - Informe en formato .txt: 
Datos del paciente.
Nombre:  Ernesto.
Apellidos: Rivera Bueno.
NHC: 368503.
NASS: 26 63514095.
Domicilio:  Calle Miguel Benitez 90.
Localidad/ Provincia: Madrid.
CP: 28016.
Datos asistenciales.
Fecha de nacimiento: 03/03/1946.
País: España.
Edad: 70 años Sexo: H.
Fecha de Ingreso: 12/12/2016.
Médico:  Ignacio Navarro Cuéllar NºCol: 28 28 70973.
Informe clínico del paciente: Paciente de 70 años de edad, minero jubilado, sin alergias medicamentosas conocidas, que presenta como antecedentes personales: accidente laboral antiguo con fracturas vertebrales y costales; intervenido de enfermedad de Dupuytren en mano derecha y by-pass iliofemoral izquierdo; Diabetes Mellitus tipo II, hipercolesterolemia e hiperuricemia; enolismo activo, fumador de 20 cigarrillos / día.
Es derivado desde Atención Primaria por presentar hematuria macroscópica postmiccional en una ocasión y microhematuria persistente posteriormente, con micciones normales.
En la exploración física presenta un buen estado general, con abdomen y genitales normales; tacto rectal compatible con adenoma de próstata grado I/IV.
En la analítica de orina destaca la existencia de 4 hematíes/ campo y 0-5 leucocitos/campo; resto de sedimento normal.
Hemograma normal; en la bioquímica destaca una glucemia de 169 mg/dl y triglicéridos de 456 mg/dl; función hepática y renal normal. PSA de 1.16 ng/ml.
Las citologías de orina son repetidamente sospechosas de malignidad.
En la placa simple de abdomen se valoran cambios degenerativos en columna lumbar y calcificaciones vasculares en ambos hipocondrios y en pelvis.
La ecografía urológica pone de manifiesto la existencia de quistes corticales simples en riñón derecho, vejiga sin alteraciones con buena capacidad y próstata con un peso de 30 g.
En la UIV se observa normofuncionalismo renal bilateral, calcificaciones sobre silueta renal derecha y uréteres arrosariados con imágenes de adición en el tercio superior de ambos uréteres, en relación a pseudodiverticulosis ureteral. El cistograma demuestra una vejiga con buena capacidad, pero paredes trabeculadas en relación a vejiga de esfuerzo. La TC abdominal es normal.
La cistoscopia descubre la existencia de pequeñas tumoraciones vesicales, realizándose resección transuretral con el resultado anatomopatológico de carcinoma urotelial superficial de vejiga.
Remitido por: Ignacio Navarro Cuéllar c/ del Abedul 5-7, 2º dcha 28036 Madrid, España E-mail: nnavcu@hotmail.com.

Ejemplo - Informe en formato .xml: lo que debes generar
<?xml version='1.0' encoding='UTF-8'?>
<MEDDOCAN>
  <TEXT>
Ejemplo - Informe en formato .txt: 
Datos del paciente.
Nombre:  <TAG TYPE="NOMBRE_SUJETO_ASISTENCIA">Ernesto</TAG>.
Apellidos: <TAG TYPE="NOMBRE_SUJETO_ASISTENCIA">Rivera Bueno</TAG>.
NHC: <TAG TYPE="ID_SUJETO_ASISTENCIA">368503</TAG>.
NASS: <TAG TYPE="ID_ASEGURAMIENTO">26 63514095</TAG>.
Domicilio:  <TAG TYPE="CALLE">Calle Miguel Benitez 90</TAG>.
Localidad/ Provincia: <TAG TYPE="TERRITORIO">Madrid</TAG>.
CP: <TAG TYPE="TERRITORIO">28016</TAG>.
Datos asistenciales.
Fecha de nacimiento: <TAG TYPE="FECHAS">03/03/1946</TAG>.
País: <TAG TYPE="PAIS">España</TAG>.
Edad: <TAG TYPE="EDAD_SUJETO_ASISTENCIA">70 años</TAG> Sexo: <TAG TYPE="SEXO_SUJETO_ASISTENCIA">H</TAG>.
Fecha de Ingreso: <TAG TYPE="FECHAS">12/12/2016</TAG>.
Médico:  <TAG TYPE="NOMBRE_PERSONAL_SANITARIO">Ignacio</TAG> <TAG TYPE="NOMBRE_PERSONAL_SANITARIO">Navarro Cuéllar</TAG> NºCol: <TAG TYPE="ID_TITULACION_PERSONAL_SANITARIO">28 28 70973</TAG>.
Informe clínico del paciente: Paciente de <TAG TYPE="EDAD_SUJETO_ASISTENCIA">70 años</TAG> de edad, minero jubilado, sin alergias medicamentosas conocidas, que presenta como antecedentes personales: accidente laboral antiguo con fracturas vertebrales y costales; intervenido de enfermedad de Dupuytren en mano derecha y by-pass iliofemoral izquierdo; Diabetes Mellitus tipo II, hipercolesterolemia e hiperuricemia; enolismo activo, fumador de 20 cigarrillos / día.
Es derivado desde Atención Primaria por presentar hematuria macroscópica postmiccional en una ocasión y microhematuria persistente posteriormente, con micciones normales.
En la exploración física presenta un buen estado general, con abdomen y genitales normales; tacto rectal compatible con adenoma de próstata grado I/IV.
En la analítica de orina destaca la existencia de 4 hematíes/ campo y 0-5 leucocitos/campo; resto de sedimento normal.
Hemograma normal; en la bioquímica destaca una glucemia de 169 mg/dl y triglicéridos de 456 mg/dl; función hepática y renal normal. PSA de 1.16 ng/ml.
Las citologías de orina son repetidamente sospechosas de malignidad.
En la placa simple de abdomen se valoran cambios degenerativos en columna lumbar y calcificaciones vasculares en ambos hipocondrios y en pelvis.
La ecografía urológica pone de manifiesto la existencia de quistes corticales simples en riñón derecho, vejiga sin alteraciones con buena capacidad y próstata con un peso de 30 g.
En la UIV se observa normofuncionalismo renal bilateral, calcificaciones sobre silueta renal derecha y uréteres arrosariados con imágenes de adición en el tercio superior de ambos uréteres, en relación a pseudodiverticulosis ureteral. El cistograma demuestra una vejiga con buena capacidad, pero paredes trabeculadas en relación a vejiga de esfuerzo. La TC abdominal es normal.
La cistoscopia descubre la existencia de pequeñas tumoraciones vesicales, realizándose resección transuretral con el resultado anatomopatológico de carcinoma urotelial superficial de vejiga.
Remitido por: <TAG TYPE="NOMBRE_PERSONAL_SANITARIO">Ignacio</TAG> <TAG TYPE="NOMBRE_PERSONAL_SANITARIO">Navarro Cuéllar</TAG> <TAG TYPE="CALLE">c/ del Abedul 5-7, 2º dcha</TAG> <TAG TYPE="TERRITORIO" >28036</TAG> <TAG TYPE="TERRITORIO" >Madrid</TAG>, <TAG TYPE="PAIS">España</TAG> E-mail: <TAG TYPE="CORREO_ELECTRONICO">nnavcu@hotmail.com</TAG>.
  </TEXT>
</MEDDOCAN>

Recordá que en ningún caso debes incluir advertencias, explicaciones ni descripciones sobre la tarea, sobre la instrucción que te he dado o sobre cuestiones de funcionamiento del modelo de lenguaje.
```

### Quality control

- Enforced presence of required sections and Spanish headings.
- Enforced length of “Informe clínico del paciente” (100–1200 words; target mean ≈375).
- Removed any outputs with refusal or meta-commentary (e.g., "No puedo cumplir esta tarea").
- Corrected formatting, punctuation, and section ordering.
- Verified MEDDOCAN entity inventory usage in the intermediate XML before converting to BRAT.

## Dataset notes

- Encoding: UTF-8 is recommended for both `.txt` and `.ann`.
- Newlines and offsets: In BRAT, character offsets are byte-based on the stored text; make sure your preprocessing preserves the exact text used for annotation.
- Multi-span entities: BRAT supports discontinuous spans (e.g., `start1 end1;start2 end2`); extend the parser accordingly if present.

## Known limitations / TODO

- Add dataset description and intended use.
- Add schema definitions (labels, relations, attributes) used in this corpus.
- Add dataset statistics (document count, entity counts, etc.).
- Provide licensing and citation details.

## License

No license file was found. Please specify a license (e.g., MIT, CC BY 4.0, or custom) to clarify usage rights.

## Citation

If you use this resource, please cite it. Provide a BibTeX entry here once a publication or DOI is available.

## Contributing

Contributions are welcome. Please open an issue to discuss substantial changes or submit a pull request with clear motivation and examples.

## Contact

- Maintainer: add name/email or GitHub handle
- Issues: use the repository issue tracker