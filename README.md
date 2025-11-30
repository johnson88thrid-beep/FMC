import pytesseract
from PIL import Image
import re

def extract_text_from_image(file_path):
    img = Image.open(file_path)
    text = pytesseract.image_to_string(img)
    return text

def extract_text_from_txt(file_path):
    with open(file_path, 'r') as f:
        return f.read()

def extract_text_from_mms(file_path):
    # Placeholder: replace with actual MMS parsing logic
    return "Payment amount $1234.56 from MMS message"

def extract_text_from_mp4(file_path):
    # Placeholder: replace with speech-to-text transcription logic
    return "This video contains payment details $789.00"

def parse_usd_amount(text):
    matches = re.findall(r'$?(d+(?:.d{2})?)', text)
    if not matches:
        raise ValueError("No USD amount found in content")
    amount = float(matches[0])
    amount_cents = int(round(amount * 100))
    return amount_cents

def generate_ach_entry(amount_cents, routing="123456789", account="987654321", name="John Doe", trace_seq=1):
    check_digit = routing[-1]
    entry = (
        "6" +
        "22" +  # transaction code: credit checking
        routing[:-1] +
        check_digit +
        account.ljust(17)[:17] +
        str(amount_cents).rjust(10, '0') +
        name.ljust(22)[:22] +
        " 0" +  # discretionary + addenda indicator
        "00000000" +  # trace number prefix (dummy)
        str(trace_seq).rjust(7, '0')
    )
    return entry

def process_media_to_ach(file_path, media_type, trace_seq=1):
    """
    Process media file of various types and return ACH payment entry string.
    Returns ACH entry string on success or raises exception on failure.
    """
    if media_type in ['jpeg', 'jpg', 'png', 'img']:
        text = extract_text_from_image(file_path)
    elif media_type == 'txt':
        text = extract_text_from_txt(file_path)
    elif media_type == 'mms':
        text = extract_text_from_mms(file_path)
    elif media_type == 'mp4':
        text = extract_text_from_mp4(file_path)
    else:
        raise ValueError(f"Unsupported media type: {media_type}")

    amount_cents = parse_usd_amount(text)
    ach_entry = generate_ach_entry(amount_cents, trace_seq=trace_seq)
    return ach_entry

# Example usage returning values from main process block
if __name__ == "__main__":
    files = [
        ("invoice.jpg", "jpeg"),
        ("payment.txt", "txt"),
        ("message.mms", "mms"),
        ("ad_video.mp4", "mp4")
    ]
    ach_entries = []
    for i, (file_path, m_type) in enumerate(files, start=1):
        try:
            ach_record = process_media_to_ach(file_path, m_type, trace_seq=i)
            ach_entries.append((file_path, ach_record))
        except Exception as e:
            ach_entries.append((file_path, f"Error: {e}"))

    # ach_entries is a list of tuples (filename, ach entry or error) returned for further processing
    for filename, entry in ach_entries:
        print(f"File: {filename}
ACH Entry or Error:
{entry}
")
