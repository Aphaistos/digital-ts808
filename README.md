# Digital-TS808 : Émulation Numérique de la Tube Screamer

Ce projet vise à modéliser mathématiquement et à synthétiser numériquement le comportement de la célèbre pédale d'overdrive **Ibanez Tube Screamer TS808**. L'objectif principal est de concevoir un prototype d'effet de traitement du signal permettant d'appliquer cette distorsion caractéristique *a posteriori* sur des pistes de guitare électrique enregistrées en DI (*Direct Input*).

Pour valider la fidélité de notre modèle, les signaux obtenus par simulation numérique sont comparés en direct avec une véritable pédale physique **TS808**, insérée dans une chaîne de test matérielle impliquant une guitare *Gibson* (micros humbuckers) et un amplificateur *Blackstar* à lampes.

---

## 1. Introduction et Problématique

Lors des sessions d'enregistrement ou de production musicale, la prise de décision concernant le grain et le gain d'une guitare est souvent irréversible si elle est effectuée directement à la prise. L'utilisation d'une pédale analogique traditionnelle fige le signal d'origine. 

La problématique de ce projet est la suivante : **Comment reproduire numériquement avec fidélité le comportement non linéaire et le filtrage fréquentiel d'un circuit analogique complexe (comme la TS808) afin de pouvoir appliquer, ajuster et automatiser cet effet thermique et dynamique *a posteriori* sur un signal numérique brut ?**

L'enjeu est de capturer les deux aspects fondamentaux qui font l'identité de cette pédale :
1. **La signature fréquentielle :** Une atténuation marquée des basses fréquences pour clarifier le mix.
2. **La dynamique de saturation :** Un écrêtage doux qui réagit fidèlement à l'amplitude du signal d'entrée (l'attaque du guitariste).

---

## 2. Hypothèses d'Étude Électronique

Pour rendre l'étude analytique du circuit accessible dans le cadre des outils théoriques de CPGE (niveau MP2I), nous posons les hypothèses simplificatrices suivantes pour le régime linéaire (petits signaux) :

### Hypothèses sur l'Amplificateur Opérationnel (AOP)
* **Idéalité :** L'AOP (généralement un JRC4558 ou TL072) est considéré comme idéal. Ses courants d'entrée sont nuls ($i_+ = 0$ et $i_- = 0$) et son gain différentiel en boucle ouverte est infini ($\mu \to \infty$).
* **Régime Linéaire :** La présence d'une boucle de rétroaction négative (chemin de retour de la sortie vers l'entrée inverseuse $V_-$) garantit le fonctionnement en régime linéaire. Par conséquent, la tension différentielle d'entrée est nulle : 
  $$\epsilon = V_+ - V_- = 0 \implies V_+ = V_-$$

### Hypothèses sur les Composants et Alimentations
* **Masse Virtuelle :** Le circuit réel utilise une alimentation asymétrique de $9\text{ V}$ et crée une tension de référence continue $V_G = 4,5\text{ V}$ (masse virtuelle) pour permettre l'oscillation du signal alternatif de la guitare. Pour l'étude de la fonction de transfert en régime harmonique, cette tension de référence est considérée comme la masse d'étude ($0\text{ V}$).
* **Comportement des Diodes :** En régime de petits signaux (tension d'entrée inférieure au seuil de conduction $V_d \approx 0,6\text{ V}$), les diodes de rétroaction sont considérées comme bloquées (circuits ouverts).
* **Simplification Fréquentielle :** La capacité de stabilisation en haute fréquence $C_f = 51\text{ pF}$ placée en parallèle dans la boucle de rétroaction possède une impédance très grande devant les résistances associées aux fréquences audio de la guitare. Elle est donc négligée ($Z_{Cf} \to \infty$, modélisée par un fil coupé) dans l'étude du pré-filtrage.

---

## 3. Modélisation Mathématique (Régime Harmonique)

En passant en notation complexe, le premier bloc (Pré-filtrage + Gain) se comporte comme un amplificateur non-inverseur. La branche connectée entre l'entrée inverseuse et la masse présente une impédance complexe série $\underline{Z}_1$ égale à :

$$\underline{Z}_1 = R_1 + \frac{1}{jC_1\omega}$$

En utilisant la formule classique du diviseur de tension, on obtient la fonction de transfert harmonique brute de l'étage de pré-saturation $\underline{H}_1(j\omega)$ :

$$\underline{H}_1(j\omega) = \frac{\underline{V}_s}{\underline{V}_e} = 1 + \frac{R_f}{\underline{Z}_1} = 1 + \frac{R_f}{R_1 + \frac{1}{jC_1\omega}}$$

En réarrangeant les termes, on met en évidence la forme canonique d'un filtre correcteur actif à action proportionnelle dérivée :

$$\underline{H}_1(j\omega) = \frac{(R_1 + R_f)C_1j\omega + 1}{R_1C_1j\omega + 1}$$

Cette équation montre clairement la cassure dans le bas du spectre, fixée par la pulsation de coupure basse $\omega_c = \frac{1}{R_1C_1}$, soit une fréquence caractéristique de :

$$f_c = \frac{1}{2\pi R_1 C_1} \approx 720\text{ Hz}$$

---

## 4. Structure de l'Émulation Numérique

L'implémentation du DSP suit le pipeline à trois composants hérité de la structure analogique :

[Entrée DI Brut] -> [Filtre IIR Passe-Haut (720 Hz)] -> [Waveshaper Non-Linéaire (Diodes)] -> [Filtre Tone Ajustable] -> [Sortie]

1. **Pré-filtrage :** Implémentation d'un filtre numérique IIR d'ordre 1 via transformée bilinéaire pour reproduire la coupure à $720\text{ Hz}$.
2. **Saturateur Non-linéaire :** Application d'une fonction de transfert statique de type *soft-clipping* simulant la paire de diodes de silicium tête-bêche :
   $$y[n] = \frac{k \cdot x[n]}{1 + |k \cdot x[n]|}$$
3. **Post-filtrage (Tone) :** Modélisation de l'égaliseur passif ajustable de la pédale par une balance de filtres contrôlée par un paramètre de position $\alpha \in [0, 1]$.

---

## 5. Environnement de Test & Comparaison

Pour valider le code de ce dépôt, le protocole expérimental s'appuie sur le matériel physique suivant :
* **Guitare :** Gibson (permettant de tester le comportement du modèle face à un haut niveau de sortie et des transitoires riches induits par les micros double bobinage).
* **Pédale de référence :** Ibanez Tube Screamer TS808 originale.
* **Amplification :** Blackstar à lampes, configuré sur un canal *Clean* transparent afin de servir de plateforme neutre pour comparer l'effet numérique et l'effet analogique direct.
