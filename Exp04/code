# pip install opencv-python pytesseract matplotlib numpy
# Also install the Tesseract OCR engine and, on Windows, set the path below.
import re
import os
import cv2
import numpy as np
import pytesseract
import matplotlib.pyplot as plt

# pytesseract.pytesseract.tesseract_cmd = r"C:\Program Files\Tesseract-OCR\tesseract.exe"

# ---------------- Knowledge Base (Indian registration format) ----------------
STATE_CODES = {
    "AP", "AR", "AS", "BR", "CG", "CH", "DD", "DL", "DN", "GA", "GJ", "HP", "HR",
    "JH", "JK", "KA", "KL", "LA", "LD", "MH", "ML", "MN", "MP", "MZ", "NL", "OD",
    "PB", "PY", "RJ", "SK", "TN", "TR", "TS", "UK", "UP", "WB",
}
PATTERN = re.compile(r"^([A-Z]{2})(\d{2})([A-Z]{1,3})(\d{4})$")   # e.g. TN09AB1234
TO_DIGIT = {"O": "0", "Q": "0", "D": "0", "I": "1", "L": "1", "Z": "2", "S": "5", "B": "8", "G": "6"}
TO_LETTER = {v: k for k, v in {"O": "0", "I": "1", "Z": "2", "S": "5", "B": "8", "G": "6"}.items()}

# ---------------- Image processing ----------------
def detect_plate(img):
    gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
    gray = cv2.bilateralFilter(gray, 11, 17, 17)
    edges = cv2.Canny(gray, 30, 200)
    cnts, _ = cv2.findContours(edges, cv2.RETR_TREE, cv2.CHAIN_APPROX_SIMPLE)
    cnts = sorted(cnts, key=cv2.contourArea, reverse=True)[:30]
    for c in cnts:
        approx = cv2.approxPolyDP(c, 0.018 * cv2.arcLength(c, True), True)
        if len(approx) == 4:
            x, y, w, h = cv2.boundingRect(approx)
            if h > 0 and 2.0 <= w / h <= 6.0 and w > 80:
                return img[y:y + h, x:x + w]
    return None

def preprocess(plate):
    g = cv2.cvtColor(plate, cv2.COLOR_BGR2GRAY)
    g = cv2.resize(g, None, fx=2, fy=2, interpolation=cv2.INTER_CUBIC)
    g = cv2.GaussianBlur(g, (5, 5), 0)
    g = cv2.createCLAHE(clipLimit=2.0, tileGridSize=(8, 8)).apply(g)
    _, th = cv2.threshold(g, 0, 255, cv2.THRESH_BINARY + cv2.THRESH_OTSU)
    return th

def read_text(proc):
    cfg = "--psm 7 -c tessedit_char_whitelist=ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
    txt = pytesseract.image_to_string(proc, config=cfg)
    return re.sub(r"[^A-Z0-9]", "", txt.upper())

# ---------------- Expert system rules ----------------
def validate(text):
    if PATTERN.match(text) and text[:2] in STATE_CODES:
        return True, text, "Valid registration format."
    n = len(text)
    if 9 <= n <= 11:                       # Rule: try positional OCR-error correction
        s = n - 8                          # series letters count
        mask = "LLDD" + "L" * s + "DDDD"
        fixed = ""
        for ch, m in zip(text, mask):
            if m == "L" and ch.isdigit():
                ch = TO_LETTER.get(ch, ch)
            elif m == "D" and ch.isalpha():
                ch = TO_DIGIT.get(ch, ch)
            fixed += ch
        if PATTERN.match(fixed) and fixed[:2] in STATE_CODES:
            return True, fixed, f"Valid after OCR correction ({text} -> {fixed})."
    return False, text, "Does not satisfy registration rules."

# ---------------- Main ----------------
def process(path):
    if not path.lower().endswith((".jpg", ".jpeg", ".png")) or not os.path.exists(path):
        print("Error: provide an existing JPG, JPEG or PNG file.")
        return
    img = cv2.imread(path)
    plate = detect_plate(img)
    if plate is None:
        print("Error: number plate could not be detected. Try another image.")
        return
    proc = preprocess(plate)
    raw = read_text(proc)
    ok, final, msg = validate(raw)

    fig, ax = plt.subplots(1, 3, figsize=(14, 4))
    ax[0].imshow(cv2.cvtColor(img, cv2.COLOR_BGR2RGB)); ax[0].set_title("Original Image")
    ax[1].imshow(cv2.cvtColor(plate, cv2.COLOR_BGR2RGB)); ax[1].set_title("Cropped Plate")
    ax[2].imshow(proc, cmap="gray"); ax[2].set_title("Preprocessed Plate")
    for a in ax: a.axis("off")
    plt.tight_layout(); plt.show()

    print("OCR text            :", raw)
    print("Registration number :", final)
    print("Validation result   :", "VALID" if ok else "INVALID", "-", msg)
    if not ok:
        print("Error: text failed validation. Please check another image.")

if __name__ == "__main__":
    while True:
        p = input("\nEnter car image path (or 'q' to quit): ").strip()
        if p.lower() == "q":
            break
        process(p)
