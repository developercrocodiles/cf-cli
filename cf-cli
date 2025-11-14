#!/usr/bin/env python
# cf-cli.py
#
# A Textual-based TUI for managing Cloudflare DNS records.
#
# Dependencies:
# pip install textual httpx
#
# Usage:
# 1. Set your Cloudflare API Token (with DNS read/edit permissions):
#    export CLOUDFLARE_API_TOKEN="your_api_token_here"
# 2. Run the script:
#    python cf-cli.py
#
# Controls:
# - Use arrow keys to navigate the tree.
# - Enter on a Zone: Load/refresh its DNS records.
# - Enter on a Record: Edit the record.
# - 'a': Add a new DNS record for the selected zone.
# - 'e': Edit the selected DNS record.
# - 'd': Delete the selected DNS record.
# - 'r': Refresh the list of zones.
# - 'q': Quit the application.

import os
import httpx
from dataclasses import dataclass

from textual.app import App, ComposeResult
from textual.widgets import Header, Footer, Tree, Log, Static, Input, Button, Label, Switch
from textual.containers import Vertical, Horizontal, Grid
from textual.screen import Screen, ModalScreen
from textual.binding import Binding
from textual.notifications import Notifier

# --- API Configuration ---
API_TOKEN = os.environ.get("CLOUDFLARE_API_TOKEN")
API_BASE_URL = "https://api.cloudflare.com/client/v4"

if not API_TOKEN:
    print("Error: CLOUDFLARE_API_TOKEN environment variable not set.")
    print("Please set it to your Cloudflare API token (with Zone.DNS permissions).")
    exit(1)

HEADERS = {
    "Authorization": f"Bearer {API_TOKEN}",
    "Content-Type": "application/json",
}

# --- Data Models (for clarity) ---
@dataclass
class DNSRecord:
    """Dataclass to hold DNS record information."""
    id: str
    zone_id: str
    name: str
    type: str
    content: str
    ttl: int
    proxied: bool

@dataclass
class Zone:
    """Dataclass to hold Zone information."""
    id: str
    name: str

# --- API Client ---
class CloudflareAPI:
    """
    A simple async API client for Cloudflare.
    Handles basic GET, POST, PUT, DELETE operations for DNS.
    """
    def __init__(self):
        self.client = httpx.AsyncClient(headers=HEADERS, base_url=API_BASE_URL, timeout=10.0)

    async def close(self):
        """Closes the HTTP client."""
        await self.client.aclose()

    async def _handle_api_error(self, e: httpx.HTTPStatusError) -> str:
        """Formats an API error message."""
        try:
            # Try to get the specific Cloudflare error message
            errors = e.response.json().get("errors", [])
            if errors:
                return f"API Error: {e.response.status_code} - {errors[0].get('message', e.response.text)}"
        except Exception:
            pass
        # Fallback
        return f"API Error: {e.response.status_code} - {e.response.text}"

    async def get_zones(self) -> list[Zone] | str:
        """Fetches all zones."""
        try:
            response = await self.client.get("/zones")
            response.raise_for_status() # Raise exception for 4xx/5xx
            zones_data = response.json().get("result", [])
            return [Zone(id=z["id"], name=z["name"]) for z in zones_data]
        except httpx.HTTPStatusError as e:
            return await self._handle_api_error(e)
        except Exception as e:
            return f"Error: {e}"

    async def get_dns_records(self, zone_id: str) -> list[DNSRecord] | str:
        """Fetches all DNS records for a given zone."""
        try:
            response = await self.client.get(f"/zones/{zone_id}/dns_records")
            response.raise_for_status()
            records_data = response.json().get("result", [])
            return [
                DNSRecord(
                    id=r["id"],
                    zone_id=r["zone_id"],
                    name=r["name"],
                    type=r["type"],
                    content=r["content"],
                    ttl=r["ttl"],
                    proxied=r.get("proxied", False), # .get for records like MX
                )
                for r in records_data
            ]
        except httpx.HTTPStatusError as e:
            return await self._handle_api_error(e)
        except Exception as e:
            return f"Error: {e}"

    async def create_dns_record(self, zone_id: str, data: dict) -> dict | str:
        """Creates a new DNS record."""
        try:
            response = await self.client.post(f"/zones/{zone_id}/dns_records", json=data)
            response.raise_for_status()
            return response.json().get("result")
        except httpx.HTTPStatusError as e:
            return await self._handle_api_error(e)
        except Exception as e:
            return f"Error: {e}"

    async def update_dns_record(self, zone_id: str, record_id: str, data: dict) -> dict | str:
        """Updates an existing DNS record."""
        try:
            # Use PUT for a full update, which is what our form provides.
            response = await self.client.put(f"/zones/{zone_id}/dns_records/{record_id}", json=data)
            response.raise_for_status()
            return response.json().get("result")
        except httpx.HTTPStatusError as e:
            return await self._handle_api_error(e)
        except Exception as e:
            return f"Error: {e}"

    async def delete_dns_record(self, zone_id: str, record_id: str) -> dict | str:
        """Deletes a DNS record."""
        try:
            response = await self.client.delete(f"/zones/{zone_id}/dns_records/{record_id}")
            response.raise_for_status()
            return response.json().get("result")
        except httpx.HTTPStatusError as e:
            return await self._handle_api_error(e)
        except Exception as e:
            return f"Error: {e}"


# --- Modals / Screens ---

class ConfirmDeleteScreen(ModalScreen[bool]):
    """Modal to confirm deletion of a DNS record."""
    def __init__(self, record_name: str, **kwargs):
        super().__init__(**kwargs)
        self.record_name = record_name

    def compose(self) -> ComposeResult:
        yield Grid(
            Label(f"Really delete {self.record_name}?", id="question"),
            Button("Delete", variant="error", id="delete"),
            Button("Cancel", variant="primary", id="cancel"),
            id="dialog",
        )

    def on_button_pressed(self, event: Button.Pressed) -> None:
        if event.button.id == "delete":
            self.dismiss(True)
        else:
            self.dismiss(False)

class RecordEditScreen(ModalScreen[dict | None]):
    """Modal screen for adding or editing a DNS record."""
    
    def __init__(self, zone_name: str, record: DNSRecord | None = None, **kwargs):
        super().__init__(**kwargs)
        self.zone_name = zone_name
        self.record = record
        self.title_text = "Edit DNS Record" if record else "Add DNS Record"

    def compose(self) -> ComposeResult:
        # Pre-fill data if editing
        record_type = self.record.type if self.record else "A"
        # The API returns the FQDN (e.g., sub.example.com).
        # For user convenience, we only show the subdomain part (e.g., 'sub')
        # or '@' for the root domain.
        if self.record:
            if self.record.name == self.zone_name:
                record_name = "@"
            else:
                record_name = self.record.name.removesuffix(f".{self.zone_name}")
        else:
            record_name = "@"

        record_content = self.record.content if self.record else ""
        record_ttl = str(self.record.ttl) if self.record else "1" # 1 = Auto
        record_proxied = self.record.proxied if self.record else True

        with Vertical(id="record-form"):
            yield Label(f"{self.title_text} for {self.zone_name}")
            yield Horizontal(
                Label("Type:", classes="form-label"),
                Input(record_type, id="input-type", placeholder="A, AAAA, CNAME, MX..."),
            )
            yield Horizontal(
                Label("Name:", classes="form-label"),
                Input(record_name, id="input-name", placeholder="@, sub, www"),
            )
            yield Horizontal(
                Label("Content:", classes="form-label"),
                Input(record_content, id="input-content", placeholder="1.2.3.4, your.cname.com"),
            )
            yield Horizontal(
                Label("TTL:", classes="form-label"),
                Input(record_ttl, id="input-ttl", placeholder="1 (Auto) or seconds"),
            )
            yield Horizontal(
                Label("Proxied:", classes="form-label"),
                Switch(value=record_proxied, id="input-proxied"),
            )
            with Horizontal(classes="form-buttons"):
                yield Button("Save", variant="success", id="save")
                yield Button("Cancel", variant="default", id="cancel")

    def on_button_pressed(self, event: Button.Pressed) -> None:
        if event.button.id == "cancel":
            self.dismiss(None)
        elif event.button.id == "save":
            # Construct the data payload
            try:
                ttl_val = int(self.query_one("#input-ttl", Input).value)
            except ValueError:
                ttl_val = 1 # Default to Auto
            
            # Convert '@' back to the zone name for the API
            record_name = self.query_one("#input-name", Input).value
            if record_name == "@":
                record_name = self.zone_name

            data = {
                "type": self.query_one("#input-type", Input).value.upper(),
                "name": record_name,
                "content": self.query_one("#input-content", Input).value,
                "ttl": ttl_val,
                "proxied": self.query_one("#input-proxied", Switch).value,
            }
            
            # Simple validation
            if not data["type"] or not data["name"] or not data["content"]:
                self.app.notify("Type, Name, and Content are required.", title="Error", severity="error")
                return

            self.dismiss(data)


# --- Main App ---

class CloudflareTUI(App):
    """A Textual TUI for Cloudflare DNS."""

    CSS_PATH = "cf-cli.tcss"
    TITLE = "Cloudflare DNS TUI"
    SUB_TITLE = "(A)dd (E)dit (D)elete (R)efresh (Q)uit"

    BINDINGS = [
        Binding("q", "quit", "Quit", priority=True),
        Binding("r", "refresh_zones", "Refresh Zones"),
        Binding("a", "add_record", "Add Record"),
        Binding("e", "edit_record", "Edit Record"),
        Binding("d", "delete_record", "Delete Record"),
        Binding("enter", "select_node", "Select", show=False),
    ]

    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.api = CloudflareAPI()
        self.tree = Tree(self.TITLE, id="tree")
        self.tree.root.expand()
        self.log_widget = Log(id="log", max_lines=100, highlight=True)
        self.notifier = Notifier(self)

    def compose(self) -> ComposeResult:
        yield Header()
        yield Vertical(
            Static("Loading zones... Press 'r' to refresh.", id="loading-static"),
            self.tree,
            self.log_widget,
        )
        yield Footer()

    async def on_mount(self) -> None:
        """Called when app starts."""
        self.log_widget.write("App mounted. Fetching zones...")
        await self.load_zones()

    async def on_quit(self) -> None:
        """Called when app quits."""
        await self.api.close()

    def notify(self, message: str, title: str = "Info", severity: str = "information"):
        """Helper to post a notification and log it."""
        self.notifier.post(title, message, severity=severity)
        log_levels = {
            "information": "",
            "warning": "[yellow]Warning:[/yellow] ",
            "error": "[red]Error:[/red] "
        }
        self.log_widget.write(f"{log_levels.get(severity, '')}{title}: {message}")


    async def load_zones(self):
        """Fetches zones and populates the tree."""
        self.tree.clear()
        self.query_one("#loading-static").display = True
        self.log_widget.write("Fetching zones from Cloudflare...")
        
        result = await self.api.get_zones()
        
        self.query_one("#loading-static").display = False
        
        if isinstance(result, str): # It's an error message
            self.notify(result, title="API Error", severity="error")
        elif isinstance(result, list):
            self.log_widget.write(f"Loaded {len(result)} zones.")
            for zone in result:
                zone_node = self.tree.root.add(zone.name, data=zone, allow_expand=True)
                zone_node.add_leaf("Loading records...", data=None) # Placeholder
        self.tree.root.expand()
        self.tree.focus()

    async def load_dns_records(self, zone_node):
        """Fetches DNS records for a specific zone node."""
        zone = zone_node.data
        if not zone or not isinstance(zone, Zone):
            return

        self.log_widget.write(f"Fetching DNS records for {zone.name}...")
        zone_node.clear() # Clear "Loading..."
        zone_node.expand()

        result = await self.api.get_dns_records(zone.id)

        if isinstance(result, str): # Error
            self.notify(result, title="API Error", severity="error")
            zone_node.add_leaf(f"[red]Error: {result}[/red]", data=None)
        elif isinstance(result, list):
            self.log_widget.write(f"Loaded {len(result)} records for {zone.name}.")
            if not result:
                zone_node.add_leaf("[i]No DNS records found.[/i]", data=None)
            
            for record in result:
                # Format: [TYPE] name -> content (Proxied/DNSOnly/TTL)
                proxy_str = ""
                if record.type in ("A", "AAAA", "CNAME"):
                    proxy_str = "Proxied" if record.proxied else "DNS Only"
                else:
                    proxy_str = f"TTL: {record.ttl}"

                label = f"[{record.type}] {record.name} -> {record.content} ({proxy_str})"
                zone_node.add_leaf(label, data=record)

    # --- Action Handlers ---

    async def action_refresh_zones(self) -> None:
        """Called by 'r' binding."""
        self.notify("Refreshing zones...")
        await self.load_zones()

    async def action_select_node(self) -> None:
        """Called by 'enter' binding. Loads records for a zone or edits a record."""
        node = self.tree.cursor_node
        if node and node.data and isinstance(node.data, Zone):
            # It's a zone node, load its records
            await self.load_dns_records(node)
        elif node and node.data and isinstance(node.data, DNSRecord):
            # It's a record node, trigger edit
            await self.action_edit_record()

    async def action_add_record(self) -> None:
        """Called by 'a' binding. Opens 'Add Record' modal."""
        node = self.tree.cursor_node
        zone_node = None

        if not node:
            self.notify("Select a zone first.", title="Error", severity="error")
            return

        if isinstance(node.data, Zone):
            zone_node = node
        elif isinstance(node.data, DNSRecord):
            zone_node = node.parent
        
        if not zone_node or not isinstance(zone_node.data, Zone):
            self.notify("Select a zone or a record within a zone first.", title="Error", severity="error")
            return
        
        zone = zone_node.data
        self.log_widget.write(f"Opening 'Add Record' dialog for {zone.name}...")

        def on_modal_dismiss(data: dict | None) -> None:
            """Callback for when the Add/Edit modal closes."""
            if data:
                self.log_widget.write(f"Attempting to create record: {data['name']}")
                # Run the async API call in the background
                self.run_worker(self.do_create_record(zone_node, data), exclusive=True)

        self.push_screen(RecordEditScreen(zone_name=zone.name), on_modal_dismiss)

    async def do_create_record(self, zone_node, data: dict):
        """Worker to create a record and refresh."""
        zone = zone_node.data
        result = await self.api.create_dns_record(zone.id, data)
        
        if isinstance(result, str): # Error
            self.notify(result, title="API Error", severity="error")
        else:
            self.notify(f"Record {data['name']} created.", title="Success")
            # Refresh the records for this zone
            await self.load_dns_records(zone_node)

    async def action_edit_record(self) -> None:
        """Called by 'e' binding. Opens 'Edit Record' modal."""
        node = self.tree.cursor_node
        
        if not node or not node.data or not isinstance(node.data, DNSRecord):
            self.notify("Select a DNS record to edit.", title="Error", severity="warning")
            return

        record = node.data
        zone_node = node.parent
        zone = zone_node.data
        self.log_widget.write(f"Opening 'Edit Record' dialog for {record.name}...")

        def on_modal_dismiss(data: dict | None) -> None:
            """Callback for when the Add/Edit modal closes."""
            if data:
                self.log_widget.write(f"Attempting to update record: {record.name}")
                self.run_worker(self.do_update_record(zone_node, record, data), exclusive=True)

        self.push_screen(RecordEditScreen(zone_name=zone.name, record=record), on_modal_dismiss)

    async def do_update_record(self, zone_node, record: DNSRecord, data: dict):
        """Worker to update a record and refresh."""
        zone = zone_node.data
        result = await self.api.update_dns_record(zone.id, record.id, data)
        
        if isinstance(result, str): # Error
            self.notify(result, title="API Error", severity="error")
        else:
            self.notify(f"Record {record.name} updated.", title="Success")
            # Refresh the records for this zone
            await self.load_dns_records(zone_node)

    async def action_delete_record(self) -> None:
        """Called by 'd' binding. Opens confirmation modal."""
        node = self.tree.cursor_node
        
        if not node or not node.data or not isinstance(node.data, DNSRecord):
            self.notify("Select a DNS record to delete.", title="Error", severity="warning")
            return
        
        record = node.data
        zone_node = node.parent
        self.log_widget.write(f"Confirming deletion of {record.name}...")

        async def on_modal_dismiss(should_delete: bool) -> None:
            """Callback for when the delete confirmation modal closes."""
            if should_delete:
                self.log_widget.write(f"Attempting to delete record: {record.name}")
                self.run_worker(self.do_delete_record(zone_node, record), exclusive=True)
            else:
                self.log_widget.write("Deletion cancelled.")

        self.push_screen(ConfirmDeleteScreen(record_name=record.name), on_modal_dismiss)

    async def do_delete_record(self, zone_node, record: DNSRecord):
        """Worker to delete a record and refresh."""
        zone = zone_node.data
        result = await self.api.delete_dns_record(zone.id, record.id)
        
        if isinstance(result, str): # Error
            self.notify(result, title="API Error", severity="error")
        else:
            self.notify(f"Record {record.name} deleted.", title="Success")
            # Refresh the records for this zone
            await self.load_dns_records(zone_node)

if __name__ == "__main__":
    app = CloudflareTUI()
    app.run()

