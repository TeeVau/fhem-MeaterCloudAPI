# MEATER Cloud API in FHEM

Bindet einen MEATER Pro über die öffentliche MEATER Cloud API in FHEM ein.
Die Lösung verwendet ausschließlich `HTTPMOD` und einige kleine Funktionen in
`99_myUtils.pm`. Sie liest Temperaturen und den aktuellen Garvorgang, zeigt die
wichtigsten Werte direkt am Device und schreibt ausgewählte Ereignisse ins
FHEM-Log. 

## Voraussetzungen

- FHEM mit `HTTPMOD`
- MEATER-Konto und ein dort sichtbarer MEATER Pro
- bekannte Geräte-ID aus der Antwort von `GET /devices`
- eine vorhandene `99_myUtils.pm`
- `use POSIX qw(strftime);` in `99_myUtils.pm`

## Zugangsdaten lokal hinterlegen

E-Mail-Adresse und Passwort werden ausschließlich im lokalen FHEM-Key-Value-
Speicher abgelegt. Die Platzhalter nur direkt in der eigenen FHEM-Installation
ersetzen und die ausgefüllten Befehle nicht veröffentlichen:

```text
{storeKeyValue("meater_email","MEATER_EMAIL")}
{storeKeyValue("meater_password","MEATER_PASSWORD")}
```


## HTTPMOD-Device

defmod MeaterCloud HTTPMOD https://public-api.cloud.meater.com/v1/devices/DEVICE_ID 60
attr MeaterCloud enableControlSet 1
attr MeaterCloud enforceGoodReadingNames 1
attr MeaterCloud timeout 10
attr MeaterCloud requestHeader01 Authorization: Bearer $sid
attr MeaterCloud requestHeader02 Accept-Language: de
attr MeaterCloud reAuthRegex "statusCode"\s*:\s*401
attr MeaterCloud sid1URL https://public-api.cloud.meater.com/v1/login
attr MeaterCloud sid1Header01 Content-Type: application/json
attr MeaterCloud sid1Data {"email":"%%meater_email%%","password":"%%meater_password%%"}
attr MeaterCloud sid1IdJSON data_token
attr MeaterCloud replacement01Regex %%meater_email%%
attr MeaterCloud replacement01Mode key
attr MeaterCloud replacement01Value meater_email
attr MeaterCloud replacement02Regex %%meater_password%%
attr MeaterCloud replacement02Mode key
attr MeaterCloud replacement02Value meater_password
attr MeaterCloud DbLogExclude .*
attr MeaterCloud room Küche
```

## Definierte Readings


```text
attr MeaterCloud reading01JSON data_id
attr MeaterCloud reading01Name deviceId
attr MeaterCloud reading02JSON data_temperature_internal
attr MeaterCloud reading02Name internalTemperature
attr MeaterCloud reading03JSON data_temperature_ambient
attr MeaterCloud reading03Name ambientTemperature
attr MeaterCloud reading04JSON data_updated_at
attr MeaterCloud reading04Name updatedAt
attr MeaterCloud reading05DeleteIfUnmatched 1
attr MeaterCloud reading05JSON data_cook_id
attr MeaterCloud reading05Name cookId
attr MeaterCloud reading06DeleteIfUnmatched 1
attr MeaterCloud reading06JSON data_cook_name
attr MeaterCloud reading06Name cookName
attr MeaterCloud reading07DeleteIfUnmatched 1
attr MeaterCloud reading07JSON data_cook_state
attr MeaterCloud reading07Name cookState
attr MeaterCloud reading08DeleteIfUnmatched 1
attr MeaterCloud reading08JSON data_cook_temperature_target
attr MeaterCloud reading08Name targetTemperature
attr MeaterCloud reading09DeleteIfUnmatched 1
attr MeaterCloud reading09JSON data_cook_temperature_peak
attr MeaterCloud reading09Name peakTemperature
attr MeaterCloud reading10DeleteIfUnmatched 1
attr MeaterCloud reading10JSON data_cook_time_elapsed
attr MeaterCloud reading10Name elapsedSeconds
attr MeaterCloud reading11DeleteIfUnmatched 1
attr MeaterCloud reading11JSON data_cook_time_remaining
attr MeaterCloud reading11Name remainingSeconds
attr MeaterCloud reading12JSON status
attr MeaterCloud reading12Name apiStatus
attr MeaterCloud reading13JSON statusCode
attr MeaterCloud reading13Name apiStatusCode
```

## myUtils-Funktionen

Die drei Funktionen in `99_myUtils.pm` einfügen. `strftime` muss dort über
`use POSIX qw(strftime);` verfügbar sein.

```perl
# Berechnet die abgeleiteten MEATER-Readings innerhalb des HTTPMOD-Updatezyklus.
sub myMeaterCloudUpdateReadings($) {
  my ($name) = @_;
  my $hash = $defs{$name};
  return 0 if (!$hash);

  my $cookState = ReadingsVal($name, "cookState", "");
  my $cookActive = $cookState =~ /^(Started|Ready For Resting|Resting|OVERCOOK!)$/ ? 1 : 0;

  my %stateDE = (
    "Not Started"        => "nicht gestartet",
    "Configured"         => "konfiguriert",
    "Started"            => "läuft",
    "Ready For Resting"  => "bereit zum Ruhen",
    "Resting"            => "ruht",
    "Slightly Underdone" => "leicht untergart",
    "Finished"           => "fertig",
    "Slightly Overdone"  => "leicht übergart",
    "OVERCOOK!"          => "ÜBERGART",
  );
  my $cookStateDE = $stateDE{$cookState}
    // ($cookState ne "" ? $cookState : "kein Garvorgang");

  my $updatedAt = ReadingsNum($name, "updatedAt", 0);
  my $dataAge = $updatedAt ? int(time() - $updatedAt) : -1;
  $dataAge = 0 if ($dataAge < 0);

  my $apiStatusCode = ReadingsNum($name, "apiStatusCode", 0);
  my $cloudState = "unavailable";
  $cloudState = "auth_error"   if ($apiStatusCode == 401);
  $cloudState = "rate_limited" if ($apiStatusCode == 429);
  $cloudState = "cloud_error"  if ($apiStatusCode >= 500);
  $cloudState = $dataAge > 180 ? "stale" : "online"
    if ($apiStatusCode == 200 && $updatedAt);

  my $target = ReadingsVal($name, "targetTemperature", "");
  my $internal = ReadingsVal($name, "internalTemperature", "");
  my $targetDelta = $target ne "" && $internal ne ""
    ? sprintf("%.1f", $target - $internal)
    : "";

  my $remainingSeconds = ReadingsVal($name, "remainingSeconds", "");
  my $remainingTime = "";
  my $estimatedFinish = "";
  if ($remainingSeconds ne "") {
    if ($remainingSeconds < 0) {
      $remainingTime = "wird berechnet";
    } else {
      my $hours = int($remainingSeconds / 3600);
      my $minutes = int(($remainingSeconds % 3600) / 60);
      $remainingTime = $hours > 0 ? "$hours h $minutes min" : "$minutes min";
      $estimatedFinish = strftime("%H:%M", localtime($updatedAt + $remainingSeconds))
        if ($remainingSeconds > 0 && $updatedAt);
    }
  }

  my $elapsedSeconds = ReadingsVal($name, "elapsedSeconds", "");
  my $cookStartedAt = $updatedAt && $elapsedSeconds ne ""
    ? strftime("%Y-%m-%d %H:%M:%S", localtime($updatedAt - $elapsedSeconds))
    : "";

  readingsBulkUpdate($hash, "cookStateDE", $cookStateDE);
  readingsBulkUpdate($hash, "dataAge", $dataAge);
  readingsBulkUpdate($hash, "cloudState", $cloudState);
  readingsBulkUpdate($hash, "targetDelta", $targetDelta);
  readingsBulkUpdate($hash, "remainingTime", $remainingTime);
  readingsBulkUpdate($hash, "estimatedFinish", $estimatedFinish);
  readingsBulkUpdate($hash, "cookStartedAt", $cookStartedAt);

  return $cookActive;
}

# Baut die kompakte mehrzeilige Anzeige für das MEATER-HTTPMOD-Device.
sub myMeaterCloudStateFormat($) {
  my ($name) = @_;

  my $cloudState = ReadingsVal($name, "cloudState", "unavailable");
  my $updatedAt = ReadingsNum($name, "updatedAt", 0);
  my $dataAge = $updatedAt ? int(time() - $updatedAt) : -1;
  $dataAge = 0 if ($dataAge < 0);
  $cloudState = "stale" if ($cloudState eq "online" && $dataAge > 180);

  my $cloudLine = "Cloud: $cloudState · Datenalter: "
    . ($dataAge >= 0 ? "$dataAge s" : "unbekannt");
  $cloudLine = "ACHTUNG: Daten veraltet · $cloudLine" if ($cloudState eq "stale");

  my $cookName = ReadingsVal($name, "cookName", "");
  my $cookStateDE = ReadingsVal($name, "cookStateDE", "kein Garvorgang");
  my $internal = ReadingsVal($name, "internalTemperature", "-");
  my $ambient = ReadingsVal($name, "ambientTemperature", "-");

  if ($cookName ne "") {
    my $target = ReadingsVal($name, "targetTemperature", "");
    my $remainingTime = ReadingsVal($name, "remainingTime", "");
    my $estimatedFinish = ReadingsVal($name, "estimatedFinish", "");

    my $text = "MEATER Pro – $cookName – $cookStateDE<br/>Kern: $internal °C";
    $text .= " · Ziel: $target °C" if ($target ne "");
    $text .= "<br/>Garraum: $ambient °C";
    $text .= "<br/>Restzeit: $remainingTime" if ($remainingTime ne "");
    $text .= " · fertig ca. $estimatedFinish Uhr" if ($estimatedFinish ne "");
    return "$text<br/>$cloudLine";
  }

  return "MEATER Pro – kein Garvorgang<br/>"
    . "Kern: $internal °C · Garraum: $ambient °C<br/>"
    . $cloudLine;
}

# Protokolliert definierte MEATER-Ereignisse höchstens einmal je Garvorgang.
sub myMeaterCloudCheckEvents($$) {
  my ($name, $eventName) = @_;
  my $eventHash = $defs{$eventName};
  return if (!$defs{$name} || !$eventHash);

  my $cookId = ReadingsVal($name, "cookId", "");
  return if ($cookId eq "");

  my $cookName = ReadingsVal($name, "cookName", "Garvorgang");
  my $cookState = ReadingsVal($name, "cookState", "");
  my $targetDelta = ReadingsVal($name, "targetDelta", "");
  my $updatedAt = ReadingsNum($name, "updatedAt", 0);
  my $dataAge = $updatedAt ? int(time() - $updatedAt) : -1;
  my $cookActive = ReadingsNum($name, "cookActive", 0);

  my @events = (
    ["targetWarnCookId",
      $cookState eq "Started"
        && $targetDelta =~ /^-?\d+(?:\.\d+)?$/
        && $targetDelta > 0 && $targetDelta <= 5,
      "$cookName: Zieltemperatur fast erreicht ($targetDelta °C verbleibend)"],
    ["readyCookId",
      $cookState eq "Ready For Resting",
      "$cookName: bereit zum Ruhen"],
    ["finishedCookId",
      $cookState eq "Finished",
      "$cookName: Garvorgang beendet"],
    ["overcookCookId",
      $cookState eq "OVERCOOK!",
      "$cookName: ÜBERGART"],
    ["staleCookId",
      $cookActive && $dataAge > 180,
      "$cookName: Cloud-Daten seit $dataAge Sekunden veraltet"],
  );

  for my $event (@events) {
    my ($reading, $condition, $message) = @$event;
    next if (!$condition || ReadingsVal($eventName, $reading, "") eq $cookId);
    Log3 $name, 3, "MEATER Cloud: $message";
    readingsSingleUpdate($eventHash, $reading, $cookId, 0);
  }
}
```

## FHEM-Attribute

```text
attr MeaterCloud userReadings cookActive {myMeaterCloudUpdateReadings($name)}
attr MeaterCloud stateFormat {myMeaterCloudStateFormat($name)}
```

## Ereignisprotokoll

Der DOIF-Aufruf bleibt bewusst kurz. Er prüft bei neuen Cloud-Daten und
zusätzlich jede Minute. Die Funktion protokolliert die folgenden Ereignisse
höchstens einmal pro `cookId` im FHEM-Log:

- höchstens 5 °C bis zur Zieltemperatur
- bereit zum Ruhen
- Garvorgang beendet
- übergart
- Cloud-Daten während eines aktiven Garvorgangs älter als 180 Sekunden

```text
defmod doif_MeaterCloudEvents DOIF ([MeaterCloud:updatedAt] or [+:01]) ({myMeaterCloudCheckEvents("MeaterCloud","doif_MeaterCloudEvents")})
attr doif_MeaterCloudEvents do always
attr doif_MeaterCloudEvents DbLogExclude .*

## Erwartetes Verhalten

- `cloudState` ist bei einer erfolgreichen Antwort `online`.
- `dataAge` zeigt das Alter der Cloud-Daten in Sekunden.
- `targetDelta` zeigt die noch fehlenden Grad bis zur Zieltemperatur.
- Eine negative Restzeit wird als `wird berechnet` dargestellt.
- Bei Daten, die älter als 180 Sekunden sind, kennzeichnet `stateFormat` die
  Anzeige deutlich als veraltet.
- Nach Ende eines Garvorgangs führt das normale Verschwinden der Cook-Readings
  nicht zu einer Störungsmeldung.


## Quellen

- [Offizielle MEATER Cloud API](https://github.com/apption-labs/meater-cloud-public-rest-api)
- [FHEM HTTPMOD-Dokumentation](https://wiki.fhem.de/wiki/HTTPMOD)
- [FHEM DOIF-Referenz](https://fhem.de/commandref_DE.html#DOIF)
