use chrono::{DateTime, NaiveDate, NaiveTime, Utc};
use serde::{Deserialize, Serialize};
use std::collections::HashSet;
use url::Url;

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
pub struct AgentConfig {
    pub app_name: String,
    pub config_version: u64,
    pub polling_interval_seconds: u64,
    pub enable_sse: bool,
    pub links: Vec<LinkItem>,
    pub scheduled_actions: Vec<ScheduledAction>,
    pub reminders: Vec<ReminderRule>,
    pub holidays: Vec<NaiveDate>,
    #[serde(default)]
    pub agent: AgentRuntimeConfig,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq, Default)]
pub struct AgentRuntimeConfig {
    #[serde(default)]
    pub autostart_default: bool,
}

impl AgentConfig {
    pub fn validate(&self) -> Result<(), String> {
        require_non_empty("app_name", &self.app_name)?;

        if self.config_version == 0 {
            return Err("config_version must be greater than 0".to_string());
        }

        if self.polling_interval_seconds == 0 {
            return Err("polling_interval_seconds must be greater than 0".to_string());
        }

        let mut link_ids = HashSet::new();
        for link in &self.links {
            link.validate()?;
            if !link_ids.insert(link.id.as_str()) {
                return Err(format!("duplicate link id: {}", link.id));
            }
        }

        let mut action_ids = HashSet::new();
        for action in &self.scheduled_actions {
            action.validate()?;
            if !action_ids.insert(action.id.as_str()) {
                return Err(format!("duplicate scheduled action id: {}", action.id));
            }
        }

        let mut reminder_ids = HashSet::new();
        for reminder in &self.reminders {
            reminder.validate()?;
            let id = reminder.id();
            if !reminder_ids.insert(id) {
                return Err(format!("duplicate reminder id: {}", id));
            }
        }

        Ok(())
    }
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
pub struct LinkItem {
    pub id: String,
    pub title: String,
    pub group: String,
    pub url: String,
}

impl LinkItem {
    pub fn validate(&self) -> Result<(), String> {
        require_non_empty("link.id", &self.id)?;
        require_non_empty("link.title", &self.title)?;
        require_non_empty("link.group", &self.group)?;
        require_valid_url("link.url", &self.url)
    }
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
pub struct ScheduledAction {
    pub id: String,
    pub title: String,
    pub action_type: ScheduledActionType,
    pub url: Option<String>,
    pub schedule: ScheduleRule,
    pub notification: Option<NotificationRule>,
}

impl ScheduledAction {
    pub fn validate(&self) -> Result<(), String> {
        require_non_empty("scheduled_action.id", &self.id)?;
        require_non_empty("scheduled_action.title", &self.title)?;
        self.schedule.validate()?;

        if let Some(notification) = &self.notification {
            notification.validate()?;
        }

        match self.action_type {
            ScheduledActionType::OpenUrl => match &self.url {
                Some(url) => require_valid_url("scheduled_action.url", url),
                None => Err("scheduled_action.url is required for OpenUrl".to_string()),
            },
        }
    }
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
pub enum ScheduledActionType {
    OpenUrl,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
#[serde(tag = "type")]
pub enum ScheduleRule {
    Daily {
        time: NaiveTime,
        weekdays: Vec<WeekdayName>,
    },
}

impl ScheduleRule {
    pub fn validate(&self) -> Result<(), String> {
        match self {
            Self::Daily { weekdays, .. } => {
                if weekdays.is_empty() {
                    return Err("schedule.weekdays must not be empty".to_string());
                }
                Ok(())
            }
        }
    }
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
pub enum WeekdayName {
    Mon,
    Tue,
    Wed,
    Thu,
    Fri,
    Sat,
    Sun,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
pub struct NotificationRule {
    pub enabled: bool,
    pub show_before_minutes: u32,
}

impl NotificationRule {
    pub fn validate(&self) -> Result<(), String> {
        if self.enabled && self.show_before_minutes == 0 {
            return Err(
                "notification.show_before_minutes must be greater than 0 when enabled".to_string(),
            );
        }
        Ok(())
    }
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
#[serde(tag = "type")]
pub enum ReminderRule {
    DayBeforeMonthDay {
        id: String,
        title: String,
        body: String,
        month_day: String,
        time: NaiveTime,
    },
    DayBeforeNthBusinessDayNextMonth {
        id: String,
        title: String,
        body: String,
        business_day_number: u8,
        time: NaiveTime,
    },
}

impl ReminderRule {
    pub fn id(&self) -> &str {
        match self {
            Self::DayBeforeMonthDay { id, .. } => id,
            Self::DayBeforeNthBusinessDayNextMonth { id, .. } => id,
        }
    }

    pub fn validate(&self) -> Result<(), String> {
        match self {
            Self::DayBeforeMonthDay {
                id,
                title,
                body,
                month_day,
                ..
            } => {
                require_non_empty("reminder.id", id)?;
                require_non_empty("reminder.title", title)?;
                require_non_empty("reminder.body", body)?;
                validate_month_day(month_day)
            }
            Self::DayBeforeNthBusinessDayNextMonth {
                id,
                title,
                body,
                business_day_number,
                ..
            } => {
                require_non_empty("reminder.id", id)?;
                require_non_empty("reminder.title", title)?;
                require_non_empty("reminder.body", body)?;

                if *business_day_number == 0 {
                    return Err("reminder.business_day_number must be greater than 0".to_string());
                }
                Ok(())
            }
        }
    }
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
pub struct NotificationMessage {
    pub id: String,
    pub title: String,
    pub body: String,
    pub severity: NotificationSeverity,
    pub created_at: DateTime<Utc>,
    pub expires_at: Option<DateTime<Utc>>,
}

impl NotificationMessage {
    pub fn validate(&self) -> Result<(), String> {
        require_non_empty("notification_message.id", &self.id)?;
        require_non_empty("notification_message.title", &self.title)?;
        require_non_empty("notification_message.body", &self.body)?;

        if let Some(expires_at) = self.expires_at {
            if expires_at <= self.created_at {
                return Err("notification_message.expires_at must be after created_at".to_string());
            }
        }

        Ok(())
    }
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
pub struct CreateNotificationRequest {
    pub title: String,
    pub body: String,
    pub severity: NotificationSeverity,
    pub expires_at: Option<DateTime<Utc>>,
}

impl CreateNotificationRequest {
    pub fn validate(&self) -> Result<(), String> {
        require_non_empty("create_notification.title", &self.title)?;
        require_non_empty("create_notification.body", &self.body)?;
        Ok(())
    }
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
pub enum NotificationSeverity {
    Info,
    Warning,
    Critical,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
pub struct AgentUpdateInfo {
    pub version: String,
    pub notes: String,
    pub download_url: String,
    pub published_at: DateTime<Utc>,
    pub sha256: String,
}

impl AgentUpdateInfo {
    pub fn validate(&self) -> Result<(), String> {
        require_non_empty("agent_update.version", &self.version)?;
        require_non_empty("agent_update.notes", &self.notes)?;
        require_valid_url("agent_update.download_url", &self.download_url)?;
        require_non_empty("agent_update.sha256", &self.sha256)
    }
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
pub struct DeviceRegistrationRequest {
    pub device: DeviceInfo,
}

impl DeviceRegistrationRequest {
    pub fn validate(&self) -> Result<(), String> {
        self.device.validate()
    }
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
pub struct DeviceHeartbeatRequest {
    pub device_id: String,
    pub hostname: String,
    pub username: String,
    pub agent_version: String,
    pub sent_at: DateTime<Utc>,
}

impl DeviceHeartbeatRequest {
    pub fn validate(&self) -> Result<(), String> {
        require_non_empty("device_heartbeat.device_id", &self.device_id)?;
        require_non_empty("device_heartbeat.hostname", &self.hostname)?;
        require_non_empty("device_heartbeat.username", &self.username)?;
        require_non_empty("device_heartbeat.agent_version", &self.agent_version)
    }
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
pub struct DeviceInfo {
    pub hostname: String,
    pub username: String,
    pub agent_version: String,
}

impl DeviceInfo {
    pub fn validate(&self) -> Result<(), String> {
        require_non_empty("device.hostname", &self.hostname)?;
        require_non_empty("device.username", &self.username)?;
        require_non_empty("device.agent_version", &self.agent_version)
    }
}

fn require_non_empty(field: &str, value: &str) -> Result<(), String> {
    if value.trim().is_empty() {
        Err(format!("{} must not be empty", field))
    } else {
        Ok(())
    }
}

fn require_valid_url(field: &str, value: &str) -> Result<(), String> {
    let parsed = Url::parse(value).map_err(|e| format!("{} is not a valid URL: {}", field, e))?;
    match parsed.scheme() {
        "http" | "https" => Ok(()),
        other => Err(format!(
            "{} must use http or https scheme, got {}",
            field, other
        )),
    }
}

fn validate_month_day(value: &str) -> Result<(), String> {
    let mut parts = value.split('-');
    let month = parts
        .next()
        .ok_or_else(|| "month_day must be in MM-DD format".to_string())?
        .parse::<u32>()
        .map_err(|_| "month_day month is invalid".to_string())?;
    let day = parts
        .next()
        .ok_or_else(|| "month_day must be in MM-DD format".to_string())?
        .parse::<u32>()
        .map_err(|_| "month_day day is invalid".to_string())?;

    if parts.next().is_some() {
        return Err("month_day must be in MM-DD format".to_string());
    }

    if NaiveDate::from_ymd_opt(2024, month, day).is_none() {
        return Err("month_day is not a valid calendar date".to_string());
    }

    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;

    fn sample_config() -> AgentConfig {
        AgentConfig {
            app_name: "DevTray".to_string(),
            config_version: 1,
            polling_interval_seconds: 30,
            enable_sse: true,
            links: vec![LinkItem {
                id: "docs".to_string(),
                title: "Project Docs".to_string(),
                group: "work".to_string(),
                url: "https://example.com/docs".to_string(),
            }],
            scheduled_actions: vec![ScheduledAction {
                id: "daily-standup".to_string(),
                title: "Open standup board".to_string(),
                action_type: ScheduledActionType::OpenUrl,
                url: Some("https://example.com/standup".to_string()),
                schedule: ScheduleRule::Daily {
                    time: NaiveTime::from_hms_opt(9, 0, 0).expect("valid time"),
                    weekdays: vec![
                        WeekdayName::Mon,
                        WeekdayName::Tue,
                        WeekdayName::Wed,
                        WeekdayName::Thu,
                        WeekdayName::Fri,
                    ],
                },
                notification: Some(NotificationRule {
                    enabled: true,
                    show_before_minutes: 10,
                }),
            }],
            reminders: vec![
                ReminderRule::DayBeforeMonthDay {
                    id: "invoice-reminder".to_string(),
                    title: "Invoice reminder".to_string(),
                    body: "Prepare monthly invoice".to_string(),
                    month_day: "05-15".to_string(),
                    time: NaiveTime::from_hms_opt(16, 30, 0).expect("valid time"),
                },
                ReminderRule::DayBeforeNthBusinessDayNextMonth {
                    id: "close-books".to_string(),
                    title: "Close books".to_string(),
                    body: "Finalize previous month".to_string(),
                    business_day_number: 3,
                    time: NaiveTime::from_hms_opt(11, 0, 0).expect("valid time"),
                },
            ],
            holidays: vec![
                NaiveDate::from_ymd_opt(2026, 1, 1).expect("valid date"),
                NaiveDate::from_ymd_opt(2026, 12, 25).expect("valid date"),
            ],
            agent: AgentRuntimeConfig {
                autostart_default: false,
            },
        }
    }

    #[test]
    fn agent_config_json_roundtrip() {
        let config = sample_config();
        config.validate().expect("valid sample config");

        let json = serde_json::to_string_pretty(&config).expect("serialize to JSON");
        let parsed: AgentConfig = serde_json::from_str(&json).expect("deserialize JSON");

        assert_eq!(parsed, config);
    }

    #[test]
    fn agent_config_yaml_roundtrip_from_example_file() {
        let yaml = include_str!("../../../docs/example-config.yaml");
        let parsed: AgentConfig = serde_yaml::from_str(yaml).expect("deserialize YAML");
        parsed.validate().expect("valid YAML config");

        let encoded = serde_yaml::to_string(&parsed).expect("serialize YAML");
        let reparsed: AgentConfig = serde_yaml::from_str(&encoded).expect("deserialize YAML again");

        assert_eq!(reparsed, parsed);
    }
}
