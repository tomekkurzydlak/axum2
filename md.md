cli

from __future__ import annotations

import argparse
import os
import sys
from pathlib import Path

from .config import load_repository_config
from .confluence import ConfluenceClient
from .manifest import load_manifest
from .models import ConfigurationError, PublishingError, ValidationError
from .plantuml import generate_plantuml
from .publish import publish_project


def _load_project_manifest(repo_root: Path, project_key: str):
    config = load_repository_config(repo_root / "confluence.targets.yaml")
    if project_key not in config.targets:
        raise ConfigurationError(f"Unknown project key: {project_key}")

    target = config.targets[project_key]
    manifest_path = repo_root / target.project_path / "manifest.yaml"
    manifest = load_manifest(project_key=project_key, path=manifest_path)
    return config, target, manifest


def cmd_validate(args: argparse.Namespace) -> int:
    repo_root = Path(args.repo_root).resolve()
    config = load_repository_config(repo_root / "confluence.targets.yaml")

    failures: list[str] = []
    for project_key, target in config.targets.items():
        manifest_path = repo_root / target.project_path / "manifest.yaml"
        try:
            load_manifest(project_key=project_key, path=manifest_path)
            print(f"[ok] {project_key}: {manifest_path}")
        except ValidationError as exc:
            failures.append(str(exc))

    if failures:
        for failure in failures:
            print(f"[error] {failure}", file=sys.stderr)
        return 1

    print("Validation completed")
    return 0


def cmd_generate(args: argparse.Namespace) -> int:
    repo_root = Path(args.repo_root).resolve()
    out_dir = Path(args.output_dir).resolve()
    out_dir.mkdir(parents=True, exist_ok=True)

    config = load_repository_config(repo_root / "confluence.targets.yaml")
    if args.project_key:
        project_keys = [args.project_key]
    else:
        project_keys = sorted(config.targets.keys())

    for project_key in project_keys:
        if project_key not in config.targets:
            raise ConfigurationError(f"Unknown project key: {project_key}")

        target = config.targets[project_key]
        manifest = load_manifest(
            project_key=project_key,
            path=repo_root / target.project_path / "manifest.yaml",
        )
        plantuml_code = generate_plantuml(manifest)

        output_path = out_dir / f"{project_key}.puml"
        output_path.write_text(plantuml_code, encoding="utf-8")
        print(f"[ok] generated {output_path}")

    return 0


def cmd_publish(args: argparse.Namespace) -> int:
    repo_root = Path(args.repo_root).resolve()
    config, target, manifest = _load_project_manifest(repo_root, args.project_key)

    token = os.environ.get("CONFLUENCE_TOKEN")
    if not token:
        raise PublishingError("CONFLUENCE_TOKEN env var is required")

    client = ConfluenceClient(
        base_url=config.confluence.base_url,
        token=token,
        username=os.environ.get("CONFLUENCE_USERNAME") or config.confluence.username,
    )

    plantuml_code = generate_plantuml(manifest)
    page_id, title = publish_project(client, config, target, manifest, plantuml_code)
    print(f"[ok] published project={args.project_key} page_id={page_id} title={title}")
    return 0


def build_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(description="UML Confluence builder")
    parser.add_argument("--repo-root", default=".", help="Repository root path")

    sub = parser.add_subparsers(dest="command", required=True)

    validate_cmd = sub.add_parser("validate", help="Validate manifests")
    validate_cmd.set_defaults(func=cmd_validate)

    generate_cmd = sub.add_parser("generate", help="Generate PlantUML files")
    generate_cmd.add_argument("--project-key", help="Generate only one project")
    generate_cmd.add_argument("--output-dir", default="build", help="Output directory")
    generate_cmd.set_defaults(func=cmd_generate)

    publish_cmd = sub.add_parser("publish", help="Publish one project to Confluence")
    publish_cmd.add_argument("--project-key", required=True, help="Project key from config")
    publish_cmd.set_defaults(func=cmd_publish)

    return parser


def main() -> None:
    parser = build_parser()
    args = parser.parse_args()

    try:
        code = args.func(args)
    except (ValidationError, ConfigurationError, PublishingError) as exc:
        print(f"[error] {exc}", file=sys.stderr)
        raise SystemExit(1)

    raise SystemExit(code)


if __name__ == "__main__":
    main()



==
config

from __future__ import annotations

from pathlib import Path
from urllib.parse import parse_qs, urlparse

import yaml

from .models import ConfigurationError, ConfluenceConfig, RepositoryConfig, TargetConfig


def _extract_page_id_from_url(url: str) -> str | None:
    parsed = urlparse(url)
    page_id = parse_qs(parsed.query).get("pageId")
    if page_id:
        return page_id[0]
    return None


def load_repository_config(path: Path) -> RepositoryConfig:
    if not path.exists():
        raise ConfigurationError(f"Missing config file: {path}")

    raw = yaml.safe_load(path.read_text(encoding="utf-8")) or {}
    conf = raw.get("confluence") or {}
    targets_raw = raw.get("targets") or {}

    base_url = (conf.get("base_url") or "").strip().rstrip("/")
    space_key = (conf.get("space_key") or "").strip()

    if not base_url or not space_key:
        raise ConfigurationError("confluence.base_url and confluence.space_key are required")

    targets: dict[str, TargetConfig] = {}
    for key, t in targets_raw.items():
        project_path = (t.get("project_path") or "").strip()
        if not project_path:
            raise ConfigurationError(f"targets.{key}.project_path is required")

        root_page_id = str(t.get("root_page_id")).strip() if t.get("root_page_id") else None
        root_page_url = (t.get("root_page_url") or "").strip() or None

        if not root_page_id and root_page_url:
            root_page_id = _extract_page_id_from_url(root_page_url)

        if not root_page_id:
            raise ConfigurationError(
                f"targets.{key} requires root_page_id or root_page_url with pageId query param"
            )

        targets[key] = TargetConfig(
            key=key,
            project_path=project_path,
            root_page_id=root_page_id,
            root_page_url=root_page_url,
            prefix=str(t.get("prefix", "1")).strip(),
            app_name=str(t.get("app_name", key)).strip(),
            page_title_template=str(
                t.get("page_title_template", "{prefix}.{index} Fizyczny model aplikacji {app_name}")
            ),
        )

    if not targets:
        raise ConfigurationError("No targets defined")

    return RepositoryConfig(
        confluence=ConfluenceConfig(
            base_url=base_url,
            space_key=space_key,
            username=(conf.get("username") or None),
        ),
        targets=targets,
    )


====

confluence

from __future__ import annotations

import re
from dataclasses import dataclass
from typing import Any

import requests

from .models import PublishingError


@dataclass(slots=True)
class Page:
    page_id: str
    title: str
    version: int


class ConfluenceClient:
    def __init__(self, base_url: str, token: str, username: str | None = None, timeout: int = 30):
        self.base_url = base_url.rstrip("/")
        self.timeout = timeout
        self.session = requests.Session()
        self.session.headers.update({"Content-Type": "application/json"})

        # Server/DC often works with Bearer; if your instance needs basic auth,
        # pass username and token-as-password and customize if needed.
        if username:
            self.session.auth = (username, token)
        else:
            self.session.headers.update({"Authorization": f"Bearer {token}"})

    def _url(self, path: str) -> str:
        return f"{self.base_url}{path}"

    def _request(self, method: str, path: str, **kwargs: Any) -> Any:
        response = self.session.request(method, self._url(path), timeout=self.timeout, **kwargs)
        if response.status_code >= 400:
            raise PublishingError(
                f"Confluence API error {response.status_code} for {method} {path}: {response.text[:500]}"
            )
        if response.text:
            return response.json()
        return {}

    def get_child_pages(self, parent_id: str) -> list[dict[str, Any]]:
        data = self._request(
            "GET",
            f"/rest/api/content/{parent_id}/child/page?limit=500&expand=version",
        )
        return data.get("results", [])

    def get_page_by_id(self, page_id: str) -> Page:
        data = self._request("GET", f"/rest/api/content/{page_id}?expand=version")
        return Page(page_id=str(data["id"]), title=data["title"], version=int(data["version"]["number"]))

    def find_page_by_label(self, label: str, space_key: str) -> Page | None:
        path = (
            "/rest/api/content/search"
            f"?cql=space={space_key}%20and%20type=page%20and%20label=%22{label}%22"
            "&limit=2&expand=version"
        )
        data = self._request("GET", path)
        results = data.get("results", [])
        if not results:
            return None
        first = results[0]
        return Page(
            page_id=str(first["id"]),
            title=first["title"],
            version=int(first["version"]["number"]),
        )

    def create_page(self, space_key: str, parent_id: str, title: str, body_storage: str) -> Page:
        payload = {
            "type": "page",
            "title": title,
            "space": {"key": space_key},
            "ancestors": [{"id": parent_id}],
            "body": {"storage": {"value": body_storage, "representation": "storage"}},
        }
        data = self._request("POST", "/rest/api/content", json=payload)
        return Page(page_id=str(data["id"]), title=data["title"], version=int(data["version"]["number"]))

    def update_page(self, page_id: str, title: str, body_storage: str, version: int) -> Page:
        payload = {
            "id": page_id,
            "type": "page",
            "title": title,
            "version": {"number": version + 1},
            "body": {"storage": {"value": body_storage, "representation": "storage"}},
        }
        data = self._request("PUT", f"/rest/api/content/{page_id}", json=payload)
        return Page(page_id=str(data["id"]), title=data["title"], version=int(data["version"]["number"]))

    def add_label(self, page_id: str, label: str) -> None:
        payload = [{"prefix": "global", "name": label}]
        self._request("POST", f"/rest/api/content/{page_id}/label", json=payload)


def first_free_index(children_titles: list[str], prefix: str) -> int:
    pattern = re.compile(rf"^{re.escape(prefix)}\.(\d+)\b")
    used: set[int] = set()
    for title in children_titles:
        match = pattern.match(title)
        if match:
            used.add(int(match.group(1)))

    index = 1
    while index in used:
        index += 1
    return index


def make_storage_body(project_key: str, title: str, plantuml_code: str) -> str:
    escaped = (
        plantuml_code.replace("&", "&amp;")
        .replace("<", "&lt;")
        .replace(">", "&gt;")
    )
    return (
        f"<h1>{title}</h1>"
        f"<p>Ta strona jest zarzadzana automatycznie przez pipeline. Project key: <code>{project_key}</code>.</p>"
        "<ac:structured-macro ac:name=\"plantuml\">"
        "<ac:plain-text-body><![CDATA["
        f"{escaped}"
        "]]></ac:plain-text-body>"
        "</ac:structured-macro>"
    )


==

manifest

from __future__ import annotations

from pathlib import Path

import yaml

from .models import Element, Group, ProjectManifest, Relation, ValidationError


ALLOWED_KINDS = {
    "actor",
    "component",
    "service",
    "database",
    "queue",
    "external_system",
    "system",
}


KIND_ALIASES = {
    "external": "external_system",
    "db": "database",
}


def _normalize_kind(kind: str) -> str:
    k = kind.strip().lower()
    return KIND_ALIASES.get(k, k)


def load_manifest(project_key: str, path: Path) -> ProjectManifest:
    if not path.exists():
        raise ValidationError(f"Missing manifest for project {project_key}: {path}")

    raw = yaml.safe_load(path.read_text(encoding="utf-8")) or {}
    title = str(raw.get("title") or project_key).strip()

    groups_raw = raw.get("groups") or []
    elements_raw = raw.get("elements") or []
    relations_raw = raw.get("relations") or []

    if not elements_raw:
        raise ValidationError(f"{project_key}: elements list is required")

    groups: list[Group] = []
    group_ids: set[str] = set()
    for idx, g in enumerate(groups_raw, start=1):
        group_id = str(g.get("id") or "").strip()
        label = str(g.get("label") or group_id).strip()
        if not group_id:
            raise ValidationError(f"{project_key}: groups[{idx}] missing id")
        if group_id in group_ids:
            raise ValidationError(f"{project_key}: duplicate group id: {group_id}")
        group_ids.add(group_id)
        groups.append(Group(id=group_id, label=label))

    elements: list[Element] = []
    element_ids: set[str] = set()
    for idx, e in enumerate(elements_raw, start=1):
        element_id = str(e.get("id") or "").strip()
        label = str(e.get("label") or element_id).strip()
        raw_kind = str(e.get("kind") or "component").strip()
        kind = _normalize_kind(raw_kind)
        group = str(e.get("group") or "").strip() or None

        if not element_id:
            raise ValidationError(f"{project_key}: elements[{idx}] missing id")
        if element_id in element_ids:
            raise ValidationError(f"{project_key}: duplicate element id: {element_id}")
        if kind not in ALLOWED_KINDS:
            allowed = ", ".join(sorted(ALLOWED_KINDS))
            raise ValidationError(
                f"{project_key}: elements[{idx}] has unsupported kind '{raw_kind}', allowed: {allowed}"
            )
        if group and group not in group_ids:
            raise ValidationError(
                f"{project_key}: elements[{idx}] uses unknown group '{group}'"
            )

        element_ids.add(element_id)
        elements.append(
            Element(
                id=element_id,
                label=label,
                kind=kind,
                group=group,
                description=str(e.get("description") or "").strip() or None,
                tags=[str(t).strip() for t in (e.get("tags") or []) if str(t).strip()],
            )
        )

    relations: list[Relation] = []
    allowed_directions = {"->", "-->", "<-", "<--", "--", "..>", "<..", ".."}
    for idx, r in enumerate(relations_raw, start=1):
        source = str(r.get("from") or "").strip()
        target = str(r.get("to") or "").strip()
        rel_type = str(r.get("type") or "association").strip()
        label = str(r.get("label") or "").strip() or None
        direction = str(r.get("direction") or "->").strip()

        if not source or not target:
            raise ValidationError(f"{project_key}: relations[{idx}] requires from/to")
        if source not in element_ids:
            raise ValidationError(
                f"{project_key}: relations[{idx}] references unknown source '{source}'"
            )
        if target not in element_ids:
            raise ValidationError(
                f"{project_key}: relations[{idx}] references unknown target '{target}'"
            )
        if direction not in allowed_directions:
            raise ValidationError(
                f"{project_key}: relations[{idx}] has unsupported direction '{direction}'"
            )

        relations.append(
            Relation(
                source=source,
                target=target,
                rel_type=rel_type,
                label=label,
                direction=direction,
            )
        )

    return ProjectManifest(
        project_key=project_key,
        title=title,
        groups=groups,
        elements=elements,
        relations=relations,
    )


==

models

from __future__ import annotations

from dataclasses import dataclass, field
from typing import Any


@dataclass(slots=True)
class Group:
    id: str
    label: str


@dataclass(slots=True)
class Element:
    id: str
    label: str
    kind: str
    group: str | None = None
    description: str | None = None
    tags: list[str] = field(default_factory=list)


@dataclass(slots=True)
class Relation:
    source: str
    target: str
    rel_type: str = "association"
    label: str | None = None
    direction: str = "->"


@dataclass(slots=True)
class ProjectManifest:
    project_key: str
    title: str
    elements: list[Element]
    relations: list[Relation]
    groups: list[Group] = field(default_factory=list)


@dataclass(slots=True)
class TargetConfig:
    key: str
    project_path: str
    root_page_id: str | None = None
    root_page_url: str | None = None
    prefix: str = "1"
    app_name: str = ""
    page_title_template: str = "{prefix}.{index} Fizyczny model aplikacji {app_name}"


@dataclass(slots=True)
class ConfluenceConfig:
    base_url: str
    space_key: str
    username: str | None = None


@dataclass(slots=True)
class RepositoryConfig:
    confluence: ConfluenceConfig
    targets: dict[str, TargetConfig]


class ValidationError(Exception):
    pass


class ConfigurationError(Exception):
    pass


class PublishingError(Exception):
    pass


JsonDict = dict[str, Any]


==

plantuml

from __future__ import annotations

from html import escape

from .models import ProjectManifest


KIND_DECLARATIONS = {
    "actor": "actor",
    "component": "component",
    "service": "component",
    "database": "database",
    "queue": "queue",
    "external_system": "cloud",
    "system": "rectangle",
}


def _alias_for(element_id: str) -> str:
    # PlantUML aliases should stay simple and predictable.
    return element_id.replace("-", "_").replace(".", "_")


def generate_plantuml(manifest: ProjectManifest) -> str:
    lines: list[str] = [
        "@startuml",
        "skinparam shadowing false",
        "skinparam linetype ortho",
        "left to right direction",
        f"title {manifest.title}",
        "",
    ]

    group_map: dict[str, list] = {g.id: [] for g in manifest.groups}
    ungrouped = []
    for element in manifest.elements:
        if element.group:
            group_map[element.group].append(element)
        else:
            ungrouped.append(element)

    for group in manifest.groups:
        lines.append(f'rectangle "{escape(group.label)}" {{')
        for element in group_map[group.id]:
            lines.extend(_element_lines(element.id, element.label, element.kind))
        lines.append("}")
        lines.append("")

    for element in ungrouped:
        lines.extend(_element_lines(element.id, element.label, element.kind))

    lines.append("")
    for relation in manifest.relations:
        src = _alias_for(relation.source)
        dst = _alias_for(relation.target)
        if relation.label:
            lines.append(f'{src} {relation.direction} {dst} : {escape(relation.label)}')
        else:
            lines.append(f"{src} {relation.direction} {dst}")

    lines.append("@enduml")
    return "\n".join(lines) + "\n"


def _element_lines(element_id: str, label: str, kind: str) -> list[str]:
    decl = KIND_DECLARATIONS.get(kind, "component")
    alias = _alias_for(element_id)
    safe_label = escape(label)
    return [f'{decl} "{safe_label}" as {alias}']

==

publish

from __future__ import annotations

import time

from .confluence import ConfluenceClient, first_free_index, make_storage_body
from .models import ProjectManifest, PublishingError, RepositoryConfig, TargetConfig


MAX_CREATE_RETRIES = 5


def publish_project(
    client: ConfluenceClient,
    config: RepositoryConfig,
    target: TargetConfig,
    manifest: ProjectManifest,
    plantuml_code: str,
) -> tuple[str, str]:
    label = f"uml-project-{manifest.project_key}"
    existing = client.find_page_by_label(label=label, space_key=config.confluence.space_key)

    if existing is not None:
        body = make_storage_body(manifest.project_key, existing.title, plantuml_code)
        client.update_page(
            page_id=existing.page_id,
            title=existing.title,
            body_storage=body,
            version=existing.version,
        )
        return existing.page_id, existing.title

    last_error: Exception | None = None
    for _ in range(MAX_CREATE_RETRIES):
        children = client.get_child_pages(parent_id=str(target.root_page_id))
        child_titles = [c["title"] for c in children]
        index = first_free_index(child_titles, target.prefix)

        title = target.page_title_template.format(
            prefix=target.prefix,
            index=index,
            app_name=target.app_name,
            project_key=target.key,
        )
        body = make_storage_body(manifest.project_key, title, plantuml_code)

        try:
            created = client.create_page(
                space_key=config.confluence.space_key,
                parent_id=str(target.root_page_id),
                title=title,
                body_storage=body,
            )
            client.add_label(created.page_id, label)
            return created.page_id, created.title
        except PublishingError as exc:
            last_error = exc
            time.sleep(1.0)

    raise PublishingError(
        f"Failed to create page for project {manifest.project_key} after {MAX_CREATE_RETRIES} attempts"
    ) from last_error

==
manifest

title: "Wspolna Architektura - Aplikacja ABC"

groups:
  - id: frontend
    label: "Frontend"
  - id: backend
    label: "Backend"

elements:
  - id: user
    label: "User"
    kind: actor

  - id: web_app
    label: "Web App"
    kind: component
    group: frontend

  - id: api
    label: "API"
    kind: service
    group: backend

  - id: main_db
    label: "Main DB"
    kind: database
    group: backend

  - id: payment_provider
    label: "Payment Provider"
    kind: external_system

relations:
  - from: user
    to: web_app
    type: interaction
    label: "uses"

  - from: web_app
    to: api
    type: sync
    label: "REST"

  - from: api
    to: main_db
    type: data
    label: "reads/writes"

  - from: api
    to: payment_provider
    type: sync
    label: "payment request"
