#[derive(Clone, Copy)]
enum SearchSortBy {
    CreationDate,
    ModificationDate,
}

impl SearchSortBy {
    fn parse(value: Option<&str>) -> actix_web::Result<Self> {
        match value
            .unwrap_or("creation_date")
            .to_ascii_lowercase()
            .as_str()
        {
            "creation_date" | "creation_date_utc" => Ok(Self::CreationDate),
            "modification_date" | "modification_date_utc" => Ok(Self::ModificationDate),
            _ => Err(actix_web::error::ErrorBadRequest(
                "sort_by must be creation_date or modification_date",
            )),
        }
    }

    fn column(self) -> &'static str {
        match self {
            Self::CreationDate => "sd.creation_date_utc",
            Self::ModificationDate => "sd.modification_date_utc",
        }
    }
}

#[derive(Clone, Copy)]
enum SortDirection {
    Asc,
    Desc,
}

impl SortDirection {
    fn parse(value: Option<&str>) -> actix_web::Result<Self> {
        match value.unwrap_or("desc").to_ascii_lowercase().as_str() {
            "asc" => Ok(Self::Asc),
            "desc" => Ok(Self::Desc),
            _ => Err(actix_web::error::ErrorBadRequest(
                "sort_direction must be asc or desc",
            )),
        }
    }

    fn sql(self) -> &'static str {
        match self {
            Self::Asc => "ASC",
            Self::Desc => "DESC",
        }
    }
}
