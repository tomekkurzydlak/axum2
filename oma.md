import re


def _without_spaces(text: str) -> str:
    return "".join(char for char in text if not char.isspace())


def _space_boundaries(text: str) -> set[int]:
    position = 0
    boundaries = set()
    for char in text:
        if char.isspace():
            boundaries.add(position)
        else:
            position += 1
    return boundaries


def recover_spaces(original: str, candidate: str) -> str:
    """Insert agreed spaces between letters; abstain on character disagreement.

    Existing whitespace is preserved. Whole tokens containing digits, addresses,
    or identifier syntax are protected, including their internal letter pairs.
    """
    characters = _without_spaces(original)
    if characters != _without_spaces(candidate):
        return original

    protected = set()
    offset = 0
    for token in re.findall(r"\S+", original):
        if (
            any(char.isdigit() for char in token)
            or any(marker in token for marker in ("@", "://", "_"))
            or re.search(r"\w\.\w", token)
        ):
            protected.update(range(offset + 1, offset + len(token)))
        offset += len(token)

    additions = _space_boundaries(candidate) - _space_boundaries(original) - protected
    additions = {
        position
        for position in additions
        if 0 < position < len(characters)
        and characters[position - 1].isalpha()
        and characters[position].isalpha()
    }
    output = []
    position = 0
    for char in original:
        if not char.isspace():
            if position in additions:
                output.append(" ")
            position += 1
        output.append(char)
    return "".join(output)


class SpaceRecoveryDecoder:
    """Wrap RapidOCR's CTC decoder without keeping predictions or text history."""

    def __init__(self, decoder, space_factor: float = 8.0):
        if not 1.0 <= space_factor <= 8.0:
            raise ValueError("space_factor must be between 1 and 8")
        if (
            not callable(getattr(decoder, "decode", None))
            or " " not in getattr(decoder, "dict", {})
            or decoder.get_ignored_tokens() != [0]
        ):
            raise RuntimeError("Space recovery requires RapidOCR's CTC decoder with blank=0")
        self.decoder = decoder
        self.space_index = decoder.dict[" "]
        self.space_factor = space_factor

    def __call__(self, predictions, return_word_box=False, **kwargs):
        baseline = self.decoder(predictions, return_word_box, **kwargs)
        if return_word_box:
            # Inserting spaces would invalidate the original word coordinates.
            return baseline

        indices = predictions.argmax(axis=2)
        probabilities = predictions.max(axis=2)
        candidates = indices.copy()
        promote = (indices == 0) & (
            predictions[:, :, self.space_index] * self.space_factor > probabilities
        )
        candidates[promote] = self.space_index
        decoded, _ = self.decoder.decode(
            candidates, probabilities, False, remove_duplicate=True
        )
        lines, words = baseline
        repaired = [
            (recover_spaces(text, candidate[0]), confidence)
            for (text, confidence), candidate in zip(lines, decoded)
        ]
        return repaired, words


def attach_space_recovery(engine) -> None:
    """Support both eager (3.7) and lazy (3.10) RapidOCR recognizer loading."""
    recognizer = getattr(engine, "text_rec", None)
    if recognizer is None:
        load_recognizer = getattr(engine, "_load_rec_model", None)
        if not callable(load_recognizer):
            raise RuntimeError("Unsupported RapidOCR recognizer initialization API")
        recognizer = load_recognizer()
    if not isinstance(recognizer.postprocess_op, SpaceRecoveryDecoder):
        recognizer.postprocess_op = SpaceRecoveryDecoder(recognizer.postprocess_op)

==
rapidocr

"""Local Docling OCR adapter"""

from threading import Lock
from typing import ClassVar, Literal

from docling.datamodel.pipeline_options import RapidOcrOptions
from docling.models.factories import get_ocr_factory

try:
    from docling.models.stages.ocr.rapid_ocr_model import RapidOcrModel
except ModuleNotFoundError:  # Older Docling module layout.
    from docling.models.rapid_ocr_model import RapidOcrModel

from app.services.rapidocr_spacing import attach_space_recovery


class SpacingRapidOcrOptions(RapidOcrOptions):
    kind: ClassVar[Literal["rapidocr_spacing"]] = "rapidocr_spacing"


class SpacingRapidOcrModel(RapidOcrModel):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        if self.enabled:
            use_rec = self.options.use_rec
            if use_rec is None:
                use_rec = self.reader.use_rec
            if use_rec:
                attach_space_recovery(self.reader)

    @classmethod
    def get_options_type(cls):
        return SpacingRapidOcrOptions


_registration_lock = Lock()


def with_space_recovery(options: RapidOcrOptions) -> SpacingRapidOcrOptions:
    """Keep every caller-supplied path/parameter and register only our option type."""
    factory = get_ocr_factory(allow_external_plugins=False)
    with _registration_lock:
        if SpacingRapidOcrOptions not in factory.classes:
            factory.register(
                SpacingRapidOcrModel,
                plugin_name="docling-service",
                plugin_module_name=__name__,
            )
    return SpacingRapidOcrOptions(**options.model_dump())

==
