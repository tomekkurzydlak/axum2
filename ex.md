import argparse
import os
import tempfile
from google.cloud import storage


# Ścieżki do key JSON dla service accountów
SRC_SA_KEY_FILE = "sa-source.json"
TGT_SA_KEY_FILE = "sa-target.json"


def copy_bucket(source_bucket_name: str, target_bucket_name: str, prefix: str = ""):
    src_client = storage.Client.from_service_account_json(SRC_SA_KEY_FILE)
    tgt_client = storage.Client.from_service_account_json(TGT_SA_KEY_FILE)

    src_bucket = src_client.bucket(source_bucket_name)
    tgt_bucket = tgt_client.bucket(target_bucket_name)

    blobs = src_client.list_blobs(source_bucket_name, prefix=prefix)

    copied = 0
    failed = 0

    for src_blob in blobs:
        blob_name = src_blob.name

        # Pomija pseudo-foldery, jeśli istnieją jako obiekty kończące się "/"
        if blob_name.endswith("/"):
            print(f"SKIP folder placeholder: {blob_name}")
            continue

        tmp_path = None

        try:
            print(f"Downloading: gs://{source_bucket_name}/{blob_name}")

            fd, tmp_path = tempfile.mkstemp()
            os.close(fd)

            src_blob.download_to_filename(tmp_path)

            print(f"Uploading:   gs://{target_bucket_name}/{blob_name}")

            tgt_blob = tgt_bucket.blob(blob_name)
            tgt_blob.upload_from_filename(tmp_path)

            copied += 1
            print(f"OK: {blob_name}")

        except Exception as e:
            failed += 1
            print(f"ERROR: {blob_name}: {e}")

        finally:
            if tmp_path and os.path.exists(tmp_path):
                os.remove(tmp_path)

    print()
    print(f"Done. Copied: {copied}, failed: {failed}")

def copy_one_file(source_bucket_name: str, target_bucket_name: str, blob_name: str):
    src_client = storage.Client.from_service_account_json(SRC_SA_KEY_FILE)
    tgt_client = storage.Client.from_service_account_json(TGT_SA_KEY_FILE)

    src_bucket = src_client.bucket(source_bucket_name)
    tgt_bucket = tgt_client.bucket(target_bucket_name)

    src_blob = src_bucket.blob(blob_name)

    if not src_blob.exists():
        raise FileNotFoundError(f"Nie znaleziono pliku: gs://{source_bucket_name}/{blob_name}")

    tmp_path = None

    try:
        fd, tmp_path = tempfile.mkstemp()
        os.close(fd)

        print(f"Downloading: gs://{source_bucket_name}/{blob_name}")
        src_blob.download_to_filename(tmp_path)

        print(f"Local tmp: {tmp_path}")
        print(f"Size local: {os.path.getsize(tmp_path)} bytes")

        print(f"Uploading:   gs://{target_bucket_name}/{blob_name}")
        tgt_blob = tgt_bucket.blob(blob_name)
        tgt_blob.upload_from_filename(tmp_path)

        print("OK")

    finally:
        if tmp_path and os.path.exists(tmp_path):
            os.remove(tmp_path)
            print(f"Deleted local tmp: {tmp_path}")

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("source_bucket", help="Source bucket name, without gs://")
    parser.add_argument("target_bucket", help="Target bucket name, without gs://")
    parser.add_argument("--prefix", default="", help="Optional prefix/path inside source bucket")

    args = parser.parse_args()

    copy_bucket(
        source_bucket_name=args.source_bucket,
        target_bucket_name=args.target_bucket,
        prefix=args.prefix,
    )


if __name__ == "__main__":
    main()
