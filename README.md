# Discord-Example-ComponentsV2-Go
# DiscordGo — Components V2 Full Documentation

An in-depth reference guide for designing rich, modern Discord bot UIs using **Components V2** with [`discordgo`](https://github.com/bwmarrin/discordgo) (bwmarrin/discordgo).

> **Converted from the discord.py version.** discordgo has no `LayoutView` class, no `.callback` attribute, and no cogs — it is a low-level, data-oriented binding to the Discord API. Every section below explains not just the syntax swap but the *architectural* difference, since a line-by-line Python→Go port would not compile or behave correctly.

---

## Table of Contents

1. [What is Components V2?](#what-is-components-v2)
2. [Requirements & Setup](#requirements--setup)
3. [Core Concept — Building a V2 Message](#core-concept--building-a-v2-message)
4. [Component Reference](#component-reference)
   - [Container](#container)
   - [TextDisplay](#textdisplay)
   - [Separator](#separator)
   - [Section](#section)
   - [Thumbnail](#thumbnail)
   - [MediaGallery](#mediagallery)
   - [ActionsRow](#actionsrow)
   - [Button](#button)
   - [SelectMenu (Dropdown)](#selectmenu-dropdown)
5. [Building Layouts](#building-layouts)
6. [Interaction Handling](#interaction-handling)
7. [Full Examples](#full-examples)
   - [Avatar Command](#example-1-avatar-command)
   - [Help Menu with Dropdown](#example-2-help-menu-with-dropdown)
   - [User Info Card](#example-3-user-info-card)
   - [Confirmation Prompt](#example-4-confirmation-prompt)
8. [Common Patterns & Tips](#common-patterns--tips)
9. [Known Limitations](#known-limitations)

---

## What is Components V2?

**Components V2** is Discord's advanced message component system. It lets bots send structured, visually polished messages using a **layout-based framework** rather than plain embeds.

### Key Differences: Classic Embeds vs. Components V2

| Feature | Classic Embeds | Components V2 |
|---|---|---|
| Layout Control | Highly Limited | Absolute (Containers, Rows) |
| Interactive Elements | Kept outside the embed | Inline inside containers |
| Image Display | Fixed Thumbnail/Image | Dynamic MediaGallery Component |
| Text Formatting | Strict field-based | Flexible TextDisplay markdown |
| Sections | Unsupported | Supported via Section + Thumbnail |

In discordgo, Components V2 has **no dedicated root class**. There is no `ui.LayoutView`. Instead:
- Every V2 component is a plain Go struct implementing the `discordgo.MessageComponent` interface (`Container`, `TextDisplay`, `Section`, `Thumbnail`, `MediaGallery`, `Separator`, `ActionsRow`, `Button`, `SelectMenu`, ...).
- You assemble a `[]discordgo.MessageComponent` slice by hand (struct literals, not method chaining).
- You opt into the system per-message by setting the `discordgo.MessageFlagsIsComponentsV2` flag. **Once a message is sent with this flag, it can't be removed from that message** — and `Content` / `Embeds` stop working on it.

Native support landed in discordgo **v0.29.0** (PR #1616), so `go get github.com/bwmarrin/discordgo@v0.29.0` or later is required.

---

## Requirements & Setup

```bash
https://go.dev/doc/install
```

**Important:** Components V2 requires **discordgo v0.29.0+**. Earlier tagged releases don't have the component types at all — check `go.mod` if you hit "undefined: discordgo.Container" errors.

### Bot Intents Configuration

```go
package main

import (
	"log"

	"github.com/bwmarrin/discordgo"
)

func main() {
	dg, err := discordgo.New("Bot " + botToken)
	if err != nil {
		log.Fatalf("invalid bot parameters: %v", err)
	}

	// Mandatory for reading prefix-style command text.
	dg.Identify.Intents = discordgo.IntentsGuilds |
		discordgo.IntentsGuildMessages |
		discordgo.IntentsMessageContent

	if err := dg.Open(); err != nil {
		log.Fatalf("cannot open session: %v", err)
	}
	defer dg.Close()

	select {} // block forever
}
```

### Essential Imports

```go
import (
	"github.com/bwmarrin/discordgo"
)
```

Unlike Python's `from discord.ui import (LayoutView, Container, Section, ...)`, every component type lives in the single `discordgo` package — there's nothing extra to import.

A tiny generic helper makes building components far less painful, since most optional fields in discordgo are pointers (`*bool`, `*int`, `*string`) rather than Python-style default `None` keyword args:

```go
func ptr[T any](v T) *T {
	return &v
}
```

We use `ptr(...)` throughout this document.

---

## Core Concept — Building a V2 Message

There's no root object to instantiate. The "root layer" is simply the top-level `[]discordgo.MessageComponent` slice you pass when sending or responding, combined with the V2 flag.

```go
container := discordgo.Container{
	AccentColor: nil, // no left border
	Components: []discordgo.MessageComponent{
		discordgo.TextDisplay{Content: "Hello, world!"},
	},
}

_, err := s.ChannelMessageSendComplex(channelID, &discordgo.MessageSend{
	Flags:      discordgo.MessageFlagsIsComponentsV2,
	Components: []discordgo.MessageComponent{container},
})
```

Responding to an interaction (slash command, button, select) works the same way, just through `InteractionRespond` instead of `ChannelMessageSendComplex`:

```go
err := s.InteractionRespond(i.Interaction, &discordgo.InteractionResponse{
	Type: discordgo.InteractionResponseChannelMessageWithSource,
	Data: &discordgo.InteractionResponseData{
		Flags:      discordgo.MessageFlagsIsComponentsV2,
		Components: []discordgo.MessageComponent{container},
	},
})
```

### There is no `timeout` parameter

Python's `LayoutView(timeout=120)` automatically disables listeners after N seconds. discordgo has no such lifecycle object — interactions aren't bound to an in-memory view instance, they arrive as stateless `InteractionCreate` events routed by `CustomID`. If you want timeout behavior, you implement it yourself, typically with `time.AfterFunc`:

```go
time.AfterFunc(120*time.Second, func() {
	// edit the message to show it expired / disable its buttons
})
```

### There is no `add_item()` / `clear_items()`

Since components are just slice elements, "adding" is `append()` and "clearing" is starting a new slice — there's no mutation API to learn:

```go
components := []discordgo.MessageComponent{}
components = append(components, discordgo.TextDisplay{Content: "Line 1"})
components = append(components, discordgo.TextDisplay{Content: "Line 2"})
```

---

## Component Reference

### Container

A **Container** is a structural box binding multiple layout elements together. It's the fundamental top-level block.

```go
container := discordgo.Container{
	ID:          0,              // optional; auto-assigned if 0
	AccentColor: ptr(0x5865F2),  // left border color, hex int. nil = no border
	Spoiler:     false,
	Components:  []discordgo.MessageComponent{ /* children */ },
}
```

#### Population

Go doesn't distinguish "constructor" vs "append" population like Python — you always build the `Components` slice, either inline or beforehand:

```go
// Inline
container := discordgo.Container{
	AccentColor: nil,
	Components: []discordgo.MessageComponent{
		discordgo.TextDisplay{Content: "Line 1"},
		discordgo.TextDisplay{Content: "Line 2"},
	},
}

// Built up first
items := []discordgo.MessageComponent{
	discordgo.TextDisplay{Content: "Line 1"},
	discordgo.TextDisplay{Content: "Line 2"},
}
container = discordgo.Container{AccentColor: nil, Components: items}
```

#### Border Accent Visuals

```go
// Explicit accent coloring (Discord Blurple)
withBorder := discordgo.Container{
	AccentColor: ptr(0x5865F2),
	Components: []discordgo.MessageComponent{
		discordgo.TextDisplay{Content: "This features a defined blue vertical border."},
	},
}

// Stripping the border entirely
clean := discordgo.Container{
	AccentColor: nil,
	Components: []discordgo.MessageComponent{
		discordgo.TextDisplay{Content: "This box has NO vertical left border."},
	},
}
```

There is no `discord.Color.blue()` helper in discordgo — colors are always plain `int` hex values (same numeric format as embed colors).

### TextDisplay

Renders **markdown text** natively inside layout elements.

```go
discordgo.TextDisplay{Content: "Your content string here"}
```

#### Markdown Formatting

```go
discordgo.TextDisplay{Content: "# Header 1"}
discordgo.TextDisplay{Content: "## Header 2"}
discordgo.TextDisplay{Content: "### Header 3"}
discordgo.TextDisplay{Content: "**Bolded Text** and *Italics* alongside __Underlines__"}
discordgo.TextDisplay{Content: "> Blockquote text block"}
discordgo.TextDisplay{Content: "`inline formatting syntax`"}
discordgo.TextDisplay{Content: "-# Tiny subtext notation"}
```

#### Variable-Driven Expressions

Go has no f-strings; use `fmt.Sprintf`:

```go
discordgo.TextDisplay{Content: fmt.Sprintf("**Target Member:** %s", member.User.Username)}
discordgo.TextDisplay{Content: fmt.Sprintf("**Ident ID:** `%s`", member.User.ID)}
discordgo.TextDisplay{Content: fmt.Sprintf("**Join Timestamp:** <t:%d:R>", member.JoinedAt.Unix())}
```

### Separator

Draws a horizontal division line between structural layouts.

```go
discordgo.Separator{
	Spacing: ptr(discordgo.SeparatorSpacingSizeSmall), // or SeparatorSpacingSizeLarge
	Divider: ptr(true),                                // false = invisible spacing gap
}
```

```go
container := discordgo.Container{
	AccentColor: nil,
	Components: []discordgo.MessageComponent{
		discordgo.TextDisplay{Content: "Block Alpha"},
		discordgo.Separator{Divider: ptr(true)},              // active division line
		discordgo.TextDisplay{Content: "Block Beta"},
		discordgo.Separator{Divider: ptr(false)},             // blank spacing gap
		discordgo.TextDisplay{Content: "Block Gamma"},
	},
}
```

Note the field is `Divider` (a `*bool`), not `visible` — semantics are inverted-ish in naming but identical in behavior (`Divider: ptr(true)` = visible line).

### Section

Arranges content horizontally: text on the left, an accessory (Thumbnail or Button) fixed on the right.

```go
discordgo.Section{
	ID:         0,
	Components: []discordgo.MessageComponent{ /* up to 3 TextDisplay items */ },
	Accessory:  discordgo.MessageComponent(nil), // Thumbnail or Button
}
```

```go
section := discordgo.Section{
	Components: []discordgo.MessageComponent{
		discordgo.TextDisplay{Content: "### Community Announcement Page"},
		discordgo.TextDisplay{Content: "Explore internal infrastructure guides updates."},
	},
	Accessory: discordgo.Thumbnail{
		Media:       discordgo.UnfurledMediaItem{URL: "https://example.com/logo.png"},
		Description: ptr("Guild Mark Logo"),
	},
}
```

### Thumbnail

A compact accessory image that can **only** live inside a Section's `Accessory` field.

```go
discordgo.Thumbnail{
	Media:       discordgo.UnfurledMediaItem{URL: "https://..."},
	Description: ptr("Alternative text"), // accessibility alt text
	Spoiler:     false,
}
```

```go
discordgo.Thumbnail{
	Media:       discordgo.UnfurledMediaItem{URL: member.User.AvatarURL("1024")},
	Description: ptr(fmt.Sprintf("User avatar portrait for %s", member.User.Username)),
}
```

**Constraint:** a Thumbnail cannot exist as a top-level or Container-level component — the API rejects it unless it's a Section's `Accessory`.

### MediaGallery

Groups multiple images into a gallery grid.

```go
gallery := discordgo.MediaGallery{
	Items: []discordgo.MediaGalleryItem{
		{
			Media:       discordgo.UnfurledMediaItem{URL: "https://example.com/photo.png"},
			Description: ptr("Alt text description"),
			Spoiler:     false,
		},
	},
}
```

#### Uploading Local Attachments

```go
gallery := discordgo.MediaGallery{
	Items: []discordgo.MediaGalleryItem{
		{Media: discordgo.UnfurledMediaItem{URL: "attachment://profile.png"}},
	},
}
```

Dispatch call, with the file attached the same way you would for a normal message:

```go
s.ChannelMessageSendComplex(channelID, &discordgo.MessageSend{
	Flags:      discordgo.MessageFlagsIsComponentsV2,
	Components: []discordgo.MessageComponent{gallery},
	Files: []*discordgo.File{
		{Name: "profile.png", ContentType: "image/png", Reader: imageBuffer},
	},
})
```

**Capacity:** up to **10 images** per MediaGallery.

### ActionsRow

A horizontal row for interactive controls (Buttons, SelectMenus). In discordgo this type is named `ActionsRow`, not `ActionRow`.

```go
row := discordgo.ActionsRow{
	Components: []discordgo.MessageComponent{
		discordgo.Button{Label: "Proceed", Style: discordgo.PrimaryButton, CustomID: "proceed"},
		discordgo.Button{Label: "Cancel", Style: discordgo.SecondaryButton, CustomID: "cancel"},
	},
}
```

**Limitation:** max **5 Buttons** OR **1 SelectMenu**. Cannot mix both.

### Button

```go
discordgo.Button{
	Label:    "Interactive Trigger",
	Style:    discordgo.PrimaryButton,
	CustomID: "unique_callback_id", // required unless it's a Link/Premium button
	URL:      "https://...",        // Link-style only; mutually exclusive with CustomID
	Emoji:    &discordgo.ComponentEmoji{Name: "🛡️"},
	Disabled: false,
}
```

#### Button Styles

| Style Constant | Visual Tint | Primary Intent |
|---|---|---|
| `discordgo.PrimaryButton` | Blue | Standard prominent workflows |
| `discordgo.SecondaryButton` | Grey | Secondary neutral utilities |
| `discordgo.SuccessButton` | Green | Successful confirmation steps |
| `discordgo.DangerButton` | Red | Destructive workflows/Deletions |
| `discordgo.LinkButton` | Grey + Arrow | External URL navigation |
| `discordgo.PremiumButton` | Blurple | Links to a purchasable SKU |

#### There is no `.callback` on a Button

Python assigns a bound method directly to `btn.callback`. discordgo has no such slot — **all** button clicks arrive as generic `InteractionCreate` events, and you match them by `CustomID` inside one central handler (see [Interaction Handling](#interaction-handling)):

```go
btn := discordgo.Button{Label: "Accept Terms", Style: discordgo.SuccessButton, CustomID: "accept_id"}

s.AddHandler(func(s *discordgo.Session, i *discordgo.InteractionCreate) {
	if i.Type != discordgo.InteractionMessageComponent {
		return
	}
	if i.MessageComponentData().CustomID != "accept_id" {
		return
	}
	s.InteractionRespond(i.Interaction, &discordgo.InteractionResponse{
		Type: discordgo.InteractionResponseChannelMessageWithSource,
		Data: &discordgo.InteractionResponseData{
			Content: "Terms validated successfully.",
			Flags:   discordgo.MessageFlagsEphemeral,
		},
	})
})
```

### SelectMenu (Dropdown)

```go
discordgo.SelectMenu{
	MenuType:    discordgo.StringSelectMenu,
	CustomID:    "select_tier_dropdown",
	Placeholder: "Identify destination hub...",
	MinValues:   ptr(1),
	MaxValues:   1,
	Options: []discordgo.SelectMenuOption{
		{Label: "Tier A", Value: "t1", Description: "First class access", Emoji: &discordgo.ComponentEmoji{Name: "🥇"}},
		{Label: "Tier B", Value: "t2", Description: "Standard class access", Emoji: &discordgo.ComponentEmoji{Name: "🥈"}},
	},
	Disabled: false,
}
```

#### Reading a Selection

```go
s.AddHandler(func(s *discordgo.Session, i *discordgo.InteractionCreate) {
	if i.Type != discordgo.InteractionMessageComponent {
		return
	}
	data := i.MessageComponentData()
	if data.CustomID != "select_tier_dropdown" {
		return
	}
	selected := data.Values[0] // discordgo.SelectMenu.Values, populated on the incoming interaction
	s.InteractionRespond(i.Interaction, &discordgo.InteractionResponse{
		Type: discordgo.InteractionResponseUpdateMessage,
		Data: &discordgo.InteractionResponseData{
			Content: fmt.Sprintf("Redirecting to: %s", selected),
		},
	})
})
```

---

## Building Layouts

### Architecture Mapping Overview

```
[]discordgo.MessageComponent (top level, sent with MessageFlagsIsComponentsV2)
└── discordgo.Container (AccentColor: nil removes left border)
    ├── discordgo.TextDisplay
    ├── discordgo.Separator
    ├── discordgo.Section
    │   ├── discordgo.TextDisplay ×1-3 (left)
    │   └── Accessory: Thumbnail | Button (right)
    ├── discordgo.MediaGallery
    │   └── Items []MediaGalleryItem
    ├── discordgo.Separator
    └── discordgo.ActionsRow
        ├── discordgo.Button
        └── discordgo.Button
```

### Component Nesting Constraints

| Parent | Permitted Children |
|---|---|
| Top level | `Container` instances only |
| `Container` | `TextDisplay`, `Separator`, `Section`, `MediaGallery`, `ActionsRow` |
| `Section` | `TextDisplay` ×1-3 (left) + `Accessory`: `Thumbnail` or `Button` (right) |
| `ActionsRow` | up to **5** `Button`s, OR exactly **1** `SelectMenu` |
| `MediaGallery` | `MediaGalleryItem` entries (up to 10) |

**Guardrail:** Containers **cannot** be nested inside other Containers. This is enforced by the Discord API, not the library — sending an invalid tree returns a 400 error.

---

## Interaction Handling

This is where discordgo diverges most from discord.py. There's no `interaction_check` method and no view state object — every interaction (slash command, button click, select) is a single global `InteractionCreate` event. You are responsible for routing and for tracking who's "allowed" to click what.

### Restricting Actions to the Command Author

Since there's no `self.author` bound to a view instance, you have two common options:

**Option A — encode the author ID into the `CustomID`:**

```go
customID := fmt.Sprintf("confirm:%s", author.ID)

s.AddHandler(func(s *discordgo.Session, i *discordgo.InteractionCreate) {
	if i.Type != discordgo.InteractionMessageComponent {
		return
	}
	parts := strings.SplitN(i.MessageComponentData().CustomID, ":", 2)
	if len(parts) != 2 || parts[0] != "confirm" {
		return
	}
	if parts[1] != i.Member.User.ID {
		s.InteractionRespond(i.Interaction, &discordgo.InteractionResponse{
			Type: discordgo.InteractionResponseChannelMessageWithSource,
			Data: &discordgo.InteractionResponseData{
				Content: "Access Denied: This dashboard belongs to another user.",
				Flags:   discordgo.MessageFlagsEphemeral,
			},
		})
		return
	}
	// proceed...
})
```

**Option B — keep an in-memory `map[string]string` of messageID → allowedUserID**, useful when you don't want the restriction visible in the `CustomID`. Guard it with a `sync.Mutex` if accessed from multiple goroutines.

### Dynamic Component Redraw

Equivalent to Python's `self.clear_items()` + `edit_message(view=self)` — in Go you just build a fresh slice and call `InteractionRespond` with `InteractionResponseUpdateMessage`:

```go
func onRefreshTrigger(s *discordgo.Session, i *discordgo.InteractionCreate) {
	updated := discordgo.Container{
		AccentColor: nil,
		Components: []discordgo.MessageComponent{
			discordgo.TextDisplay{Content: "## State Updated Successfully!"},
			discordgo.TextDisplay{Content: "System properties have altered configuration states."},
		},
	}

	s.InteractionRespond(i.Interaction, &discordgo.InteractionResponse{
		Type: discordgo.InteractionResponseUpdateMessage,
		Data: &discordgo.InteractionResponseData{
			Flags:      discordgo.MessageFlagsIsComponentsV2,
			Components: []discordgo.MessageComponent{updated},
		},
	})
}
```

---

## Full Examples

These mirror the four examples from the Python doc. Since discordgo has no cog system, each example is a self-contained function you register with `AddHandler` and/or `ApplicationCommandCreate`, following the same routing pattern the [official discordgo components example](https://github.com/bwmarrin/discordgo/tree/master/examples/components) uses.

### Example 1: Avatar Command

```go
package main

import (
	"bytes"
	"fmt"
	"io"
	"net/http"

	"github.com/bwmarrin/discordgo"
)

func avatarCommand(s *discordgo.Session, i *discordgo.InteractionCreate) {
	member := i.Member
	if opt, ok := optionByName(i, "user"); ok {
		member = &discordgo.Member{User: opt.UserValue(s)}
	}
	user := member.User

	pngURL := user.AvatarURL("1024") // discordgo picks gif automatically if animated
	jpgURL := avatarURLWithFormat(user, "jpg", 1024)
	filename := "avatar.png"
	if isAnimatedAvatar(user) {
		filename = "avatar.gif"
	}

	resp, err := http.Get(pngURL)
	if err != nil {
		respondError(s, i, "Couldn't fetch that avatar.")
		return
	}
	defer resp.Body.Close()
	data, _ := io.ReadAll(resp.Body)

	gallery := discordgo.MediaGallery{
		Items: []discordgo.MediaGalleryItem{
			{Media: discordgo.UnfurledMediaItem{URL: "attachment://" + filename}},
		},
	}

	buttonRow := discordgo.ActionsRow{
		Components: []discordgo.MessageComponent{
			discordgo.Button{Label: "PNG Asset", Style: discordgo.LinkButton, URL: pngURL},
			discordgo.Button{Label: "JPG Asset", Style: discordgo.LinkButton, URL: jpgURL},
		},
	}

	container := discordgo.Container{
		AccentColor: nil,
		Components: []discordgo.MessageComponent{
			discordgo.TextDisplay{Content: fmt.Sprintf("# Profile: %s", user.Username)},
			discordgo.Separator{Divider: ptr(true)},
			discordgo.TextDisplay{Content: fmt.Sprintf("**System Identification:** `%s`", user.ID)},
			gallery,
			discordgo.Separator{Divider: ptr(true)},
			buttonRow,
		},
	}

	s.InteractionRespond(i.Interaction, &discordgo.InteractionResponse{
		Type: discordgo.InteractionResponseChannelMessageWithSource,
		Data: &discordgo.InteractionResponseData{
			Flags:      discordgo.MessageFlagsIsComponentsV2,
			Components: []discordgo.MessageComponent{container},
			Files: []*discordgo.File{
				{Name: filename, ContentType: "image/png", Reader: bytes.NewReader(data)},
			},
		},
	})
}

func isAnimatedAvatar(u *discordgo.User) bool {
	return len(u.Avatar) > 2 && u.Avatar[:2] == "a_"
}

func avatarURLWithFormat(u *discordgo.User, format string, size int) string {
	return fmt.Sprintf("https://cdn.discordapp.com/avatars/%s/%s.%s?size=%d", u.ID, u.Avatar, format, size)
}
```

> `ptr`, `optionByName`, and `respondError` are small helpers — see [Common Patterns & Tips](#common-patterns--tips).

### Example 2: Help Menu with Dropdown

```go
package main

import (
	"fmt"
	"strings"

	"github.com/bwmarrin/discordgo"
)

var modules = map[string][]string{
	"Admin":  {"`ban`", "`kick`", "`clear`"},
	"Public": {"`ping`", "`info`", "`help`"},
}

func helpDropdownOptions(selected string) []discordgo.SelectMenuOption {
	opts := make([]discordgo.SelectMenuOption, 0, len(modules))
	for name := range modules {
		opts = append(opts, discordgo.SelectMenuOption{
			Label:   name,
			Value:   name,
			Default: name == selected,
		})
	}
	return opts
}

func helpContainer(botAvatarURL string, selected string) discordgo.Container {
	if selected == "" {
		section := discordgo.Section{
			Components: []discordgo.MessageComponent{
				discordgo.TextDisplay{Content: "### 🛠️ Help Desk Matrix"},
				discordgo.TextDisplay{Content: "Toggle dropdown menus to access specific module listings."},
			},
			Accessory: discordgo.Thumbnail{
				Media:       discordgo.UnfurledMediaItem{URL: botAvatarURL},
				Description: ptr("Core Client Visual"),
			},
		}
		return discordgo.Container{
			AccentColor: nil,
			Components: []discordgo.MessageComponent{
				section,
				discordgo.Separator{Divider: ptr(true)},
				discordgo.ActionsRow{Components: []discordgo.MessageComponent{
					discordgo.SelectMenu{
						MenuType:    discordgo.StringSelectMenu,
						CustomID:    "help_dropdown",
						Placeholder: "Browse system tools modules...",
						Options:     helpDropdownOptions(""),
					},
				}},
			},
		}
	}

	return discordgo.Container{
		AccentColor: nil,
		Components: []discordgo.MessageComponent{
			discordgo.TextDisplay{Content: fmt.Sprintf("### 📂 Module: %s", selected)},
			discordgo.TextDisplay{Content: strings.Join(modules[selected], " ")},
			discordgo.Separator{Divider: ptr(true)},
			discordgo.ActionsRow{Components: []discordgo.MessageComponent{
				discordgo.SelectMenu{
					MenuType:    discordgo.StringSelectMenu,
					CustomID:    "help_dropdown",
					Placeholder: "Select another category...",
					Options:     helpDropdownOptions(selected),
				},
			}},
		},
	}
}

func helpCommand(s *discordgo.Session, i *discordgo.InteractionCreate) {
	container := helpContainer(s.State.User.AvatarURL("256"), "")
	s.InteractionRespond(i.Interaction, &discordgo.InteractionResponse{
		Type: discordgo.InteractionResponseChannelMessageWithSource,
		Data: &discordgo.InteractionResponseData{
			Flags:      discordgo.MessageFlagsIsComponentsV2,
			Components: []discordgo.MessageComponent{container},
		},
	})
}

func helpDropdownHandler(s *discordgo.Session, i *discordgo.InteractionCreate) {
	if i.Type != discordgo.InteractionMessageComponent {
		return
	}
	data := i.MessageComponentData()
	if data.CustomID != "help_dropdown" {
		return
	}
	// The original message author is whoever the interaction was created for;
	// discordgo doesn't track this automatically, so restrict via the message's
	// interaction metadata or your own map if this needs to be author-locked.
	selected := data.Values[0]
	container := helpContainer(s.State.User.AvatarURL("256"), selected)

	s.InteractionRespond(i.Interaction, &discordgo.InteractionResponse{
		Type: discordgo.InteractionResponseUpdateMessage,
		Data: &discordgo.InteractionResponseData{
			Flags:      discordgo.MessageFlagsIsComponentsV2,
			Components: []discordgo.MessageComponent{container},
		},
	})
}

// Registration, typically done once at startup:
//   s.AddHandler(helpDropdownHandler)
//   and helpCommand wired up as the /help slash command handler.
```

### Example 3: User Info Card

```go
package main

import (
	"fmt"

	"github.com/bwmarrin/discordgo"
)

func userInfoCommand(s *discordgo.Session, i *discordgo.InteractionCreate) {
	member := i.Member
	if opt, ok := optionByName(i, "member"); ok {
		member = opt.UserValue(s) // resolve to a discordgo.Member in a real bot via GuildMember
	}
	user := member.User

	section := discordgo.Section{
		Components: []discordgo.MessageComponent{
			discordgo.TextDisplay{Content: fmt.Sprintf("# Summary Card: %s", user.Username)},
			discordgo.TextDisplay{Content: fmt.Sprintf("**Account Handle:** %s", user.Username)},
		},
		Accessory: discordgo.Thumbnail{
			Media:       discordgo.UnfurledMediaItem{URL: user.AvatarURL("1024")},
			Description: ptr("Target Portrait"),
		},
	}

	// discordgo has no discord.Color.value shortcut — resolve the member's
	// top-role color yourself, or leave nil for the default (no border).
	accent := topRoleColor(s, i.GuildID, member)

	container := discordgo.Container{
		AccentColor: accent,
		Components: []discordgo.MessageComponent{
			section,
			discordgo.Separator{Divider: ptr(true)},
			discordgo.TextDisplay{Content: fmt.Sprintf("**Join Timestamp:** <t:%d:D>", member.JoinedAt.Unix())},
		},
	}

	s.InteractionRespond(i.Interaction, &discordgo.InteractionResponse{
		Type: discordgo.InteractionResponseChannelMessageWithSource,
		Data: &discordgo.InteractionResponseData{
			Flags:      discordgo.MessageFlagsIsComponentsV2,
			Components: []discordgo.MessageComponent{container},
		},
	})
}

// topRoleColor walks the guild's roles and returns the color of the member's
// highest colored role, or nil if they have the default color.
func topRoleColor(s *discordgo.Session, guildID string, member *discordgo.Member) *int {
	guild, err := s.State.Guild(guildID)
	if err != nil {
		return nil
	}
	roleColor := map[string]int{}
	for _, r := range guild.Roles {
		roleColor[r.ID] = r.Color
	}
	best := 0
	for _, rid := range member.Roles {
		if c := roleColor[rid]; c != 0 {
			best = c
		}
	}
	if best == 0 {
		return nil
	}
	return &best
}
```

### Example 4: Confirmation Prompt

```go
package main

import (
	"fmt"
	"time"

	"github.com/bwmarrin/discordgo"
)

func confirmationPrompt(s *discordgo.Session, i *discordgo.InteractionCreate, description string) {
	authorID := i.Member.User.ID
	yesID := "confirm_yes:" + authorID
	noID := "confirm_no:" + authorID

	container := discordgo.Container{
		AccentColor: ptr(0xF1C40F), // gold, warning indicator
		Components: []discordgo.MessageComponent{
			discordgo.TextDisplay{Content: "## ⚠️ System Authorization Gateway Request"},
			discordgo.Separator{Divider: ptr(true)},
			discordgo.TextDisplay{Content: fmt.Sprintf("Pending Operation: **%s**", description)},
			discordgo.Separator{Divider: ptr(true)},
			discordgo.ActionsRow{Components: []discordgo.MessageComponent{
				discordgo.Button{Label: "Confirm Operation", Style: discordgo.SuccessButton, CustomID: yesID},
				discordgo.Button{Label: "Abort Operation", Style: discordgo.DangerButton, CustomID: noID},
			}},
		},
	}

	s.InteractionRespond(i.Interaction, &discordgo.InteractionResponse{
		Type: discordgo.InteractionResponseChannelMessageWithSource,
		Data: &discordgo.InteractionResponseData{
			Flags:      discordgo.MessageFlagsIsComponentsV2,
			Components: []discordgo.MessageComponent{container},
		},
	})

	// Python's LayoutView(timeout=30) is emulated manually:
	time.AfterFunc(30*time.Second, func() {
		msg, err := s.InteractionResponse(i.Interaction)
		if err != nil {
			return // already resolved/edited
		}
		expired := discordgo.Container{
			AccentColor: nil,
			Components: []discordgo.MessageComponent{
				discordgo.TextDisplay{Content: "## ⌛ System Notice: Request expired"},
			},
		}
		s.ChannelMessageEditComplex(&discordgo.MessageEdit{
			Channel:    msg.ChannelID,
			ID:         msg.ID,
			Flags:      discordgo.MessageFlagsIsComponentsV2,
			Components: &[]discordgo.MessageComponent{expired},
		})
	})
}

func confirmationHandler(s *discordgo.Session, i *discordgo.InteractionCreate) {
	if i.Type != discordgo.InteractionMessageComponent {
		return
	}
	customID := i.MessageComponentData().CustomID

	switch {
	case strings.HasPrefix(customID, "confirm_yes:"):
		if strings.TrimPrefix(customID, "confirm_yes:") != i.Member.User.ID {
			return // not the original author
		}
		respondResolved(s, i, "## ✅ System Notice: Executed successfully")
	case strings.HasPrefix(customID, "confirm_no:"):
		if strings.TrimPrefix(customID, "confirm_no:") != i.Member.User.ID {
			return
		}
		respondResolved(s, i, "## ❌ System Notice: Operation Aborted")
	}
}

func respondResolved(s *discordgo.Session, i *discordgo.InteractionCreate, text string) {
	container := discordgo.Container{
		AccentColor: nil,
		Components:  []discordgo.MessageComponent{discordgo.TextDisplay{Content: text}},
	}
	s.InteractionRespond(i.Interaction, &discordgo.InteractionResponse{
		Type: discordgo.InteractionResponseUpdateMessage,
		Data: &discordgo.InteractionResponseData{
			Flags:      discordgo.MessageFlagsIsComponentsV2,
			Components: []discordgo.MessageComponent{container},
		},
	})
}
```

---

## Common Patterns & Tips

### Clearing the Left Border

Set `AccentColor: nil` on the `Container` — same idea as Python's `accent_color=None`, just spelled as a nil pointer instead of `None`.

### No Persistent Callback Re-attachment Needed

Python requires you to remember to reassign `.callback` every time you rebuild a dropdown. In discordgo there's nothing to reassign — routing happens centrally by `CustomID`, so as long as your `CustomID` strings stay consistent (or you parse them consistently), a freshly-built `SelectMenu`/`Button` "just works" the next time an interaction with that ID arrives. The tradeoff: **you** must design your `CustomID` scheme carefully (this doc uses `action:argument` style IDs throughout).

### Disabling Controls After Interaction

```go
func onButtonClick(s *discordgo.Session, i *discordgo.InteractionCreate) {
	haltBtn := discordgo.Button{
		Label:    "Session Resolved",
		Style:    discordgo.SecondaryButton,
		CustomID: "resolved_noop",
		Disabled: true,
	}
	container := discordgo.Container{
		AccentColor: nil,
		Components: []discordgo.MessageComponent{
			discordgo.TextDisplay{Content: "Task complete."},
			discordgo.ActionsRow{Components: []discordgo.MessageComponent{haltBtn}},
		},
	}
	s.InteractionRespond(i.Interaction, &discordgo.InteractionResponse{
		Type: discordgo.InteractionResponseUpdateMessage,
		Data: &discordgo.InteractionResponseData{
			Flags:      discordgo.MessageFlagsIsComponentsV2,
			Components: []discordgo.MessageComponent{container},
		},
	})
}
```

### Helper: reading a slash-command option

```go
func optionByName(i *discordgo.InteractionCreate, name string) (*discordgo.ApplicationCommandInteractionDataOption, bool) {
	for _, opt := range i.ApplicationCommandData().Options {
		if opt.Name == name {
			return opt, true
		}
	}
	return nil, false
}

func respondError(s *discordgo.Session, i *discordgo.InteractionCreate, msg string) {
	s.InteractionRespond(i.Interaction, &discordgo.InteractionResponse{
		Type: discordgo.InteractionResponseChannelMessageWithSource,
		Data: &discordgo.InteractionResponseData{
			Content: msg,
			Flags:   discordgo.MessageFlagsEphemeral,
		},
	})
}
```

### No Cogs — organize by files and a router map instead

discordgo has no `commands.Cog` / `bot.add_cog()` equivalent. The idiomatic pattern is:
1. Register slash commands once via `s.ApplicationCommandCreate(...)`.
2. Register a single `InteractionCreate` handler that switches on `i.ApplicationCommandData().Name` (for commands) or `i.MessageComponentData().CustomID` prefix (for buttons/selects).
3. Split the handler functions across regular Go files/packages for organization — there's no framework-level grouping construct.

```go
commandHandlers := map[string]func(s *discordgo.Session, i *discordgo.InteractionCreate){
	"avatar":   avatarCommand,
	"help":     helpCommand,
	"userinfo": userInfoCommand,
}

s.AddHandler(func(s *discordgo.Session, i *discordgo.InteractionCreate) {
	switch i.Type {
	case discordgo.InteractionApplicationCommand:
		if h, ok := commandHandlers[i.ApplicationCommandData().Name]; ok {
			h(s, i)
		}
	case discordgo.InteractionMessageComponent:
		customID := i.MessageComponentData().CustomID
		switch {
		case customID == "help_dropdown":
			helpDropdownHandler(s, i)
		case strings.HasPrefix(customID, "confirm_"):
			confirmationHandler(s, i)
		}
	}
})
```

---

## Known Limitations

These are Discord API-level rules, so they apply identically regardless of language/library:

| Rule | Restriction |
|---|---|
| **Zero Container Nesting** | You cannot place a `Container` inside another `Container`. |
| **Section-Confined Thumbnails** | `Thumbnail` must populate a `Section`'s `Accessory` field. |
| **ActionsRow Allocation Ceiling** | Max **5** `Button`s OR **1** `SelectMenu`, never both. |
| **MediaGallery Volume Constraint** | Up to **10** images per gallery. |
| **No Mix-and-Match** | Components V2 (`MessageFlagsIsComponentsV2`) cannot be combined with classic `Content`/`Embeds` in the same message. |

discordgo-specific notes:

| Rule | Restriction |
|---|---|
| **Version gate** | Components V2 types don't exist before discordgo **v0.29.0**. |
| **No view lifecycle** | No automatic timeouts, no automatic "disable on expiry" — implement with `time.AfterFunc` if needed. |
| **No callback binding** | All interactions funnel through `InteractionCreate`; you route by `CustomID` yourself. |

---

**Tip:** Test layouts on a real dev server. Struct literals compile even when a field combination is invalid at the Discord API level (e.g. a `Thumbnail` outside a `Section`) — the error only surfaces at runtime as an HTTP 400 from `InteractionRespond`/`ChannelMessageSendComplex`.

---

# Author

Pepsi
*(Go/discordgo conversion)*
