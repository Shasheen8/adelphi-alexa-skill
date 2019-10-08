# adelphi-alexa-skill

import logging

from flask import Flask
from flask_ask import Ask, statement, question, convert

import pandas as pd
import arrow
import re
import requests
from lxml import html
from fuzzywuzzy import fuzz
from datetime import timedelta

app = Flask(__name__)
ask = Ask(app, '/')
log = logging.getLogger()

ACADEMIC_CALENDAR = 'https://registrar.adelphi.edu/academic-calendar/'


@ask.launch
def launch():
    speech = "Welcome to Adelphi University's Academic Calendar"
    reprompt = "You can ask when certain events happen, or what is " + \
        "happening on a certain day. What do you want to know?"
    return question(speech).reprompt(reprompt)


@ask.intent("EventIntent")
def get_event(event):
    schedule = _get_academic_calendar()

    schedule['Similarity'] = schedule.apply(
        lambda row: fuzz.token_sort_ratio(row['Event'], event), axis=1)

    events = schedule[schedule['Similarity'] >= 70].sort_values(
        by=['Similarity'], ascending=False)

    if not events.empty:
        eventname = events.iloc[0]['Event']
        date = events.iloc[0]['daterange']

        day = date.format('MMMM Do')

        return statement(f"{eventname} is happening on {day}")
    else:
        return statement(f"Couldn't find an event called {event}")


@ask.intent("SemesterIntent")
def get_events_for_semester(semester):
    log.debug('Started get_events_for_semester')
    log.debug(f"Got {semester} of type {type(semester)}")
    semregex = re.compile("(?P<year>\d{4})-(?P<season>WI|SP|SU|FA)")

    gr = semregex.match(semester)

    log.debug(f"Got regex match {gr}")
    if gr is None:
        # Semester may sometimes match intents meant for get_event_on_day,
        # as they both use the same slot type - AMAZON.DATE
        # So before failing let's attempt to convert it to a date and pass
        # it to the correct function
        date = convert.to_date(semester)
        if date is not None:
            return get_event_on_day(date)
        else:
            log.debug(f"Could not make sense of given date in" +
                      f"get_events_for_semester")
            return statement(f"I didn't understand that")

    gd = gr.groupdict()

    if gd['season'] == 'WI':
        return statement(f"There is no winter semester, you can instead try" +
                         f"Fall {gd['year']} or Spring {gd['year']+1}")

    search_term = None
    if gd['season'] == 'SP':
        search_term = f"Spring {gd['year']}"
    elif gd['season'] == 'SU':
        search_term = f"Summer {gd['year']}"
    elif gd['season'] == 'FA':
        search_term = f"Fall {gd['year']}"

    if search_term is None:
        log.info("Couldn't find the season. This should never happen.")
        return statement("I didn't understand that")

    schedule = _get_academic_calendar()

    events = schedule[schedule['Semester'] == search_term]

    resp = f"There are {len(events)} events in the given semester. They are: " + \
        ', '.join([f"{r.Event} on {r.Date}" for r in events.itertuples()])
    return statement(resp)


@ask.intent('DateIntent', convert={'date': 'day'})
def get_event_on_day(date):
    log.debug('Started get_event_on_day')
    log.debug(f"Got {date} of type {type(date)}")
    if date is None:
        log.debug("Couldn't parse date")
        return statement("Sorry, I didn't understand that")

    # Convert day to Arrow for use in further operations (Probably not
    # required)
    # day = arrow.Arrow.fromdate(date)
    day = arrow.get(date)

    schedule = _get_academic_calendar()

    matchedrow = None

    # This loop assumes that 2 events can't be scheduled on the same
    # day. If that assumption turns out to be false, change how we gather
    # the rows and go through the whole list.
    log.debug('Entering schedule loop')

    for e in schedule.itertuples():
        d = e.daterange
        if type(d) == list:
            # Check if any of the days in a daterange is the one we're
            # looking for
            if any((timedelta(0) <= day-dr < timedelta(1) for dr in d)):
                matchedrow = e
                break
        elif type(d) == arrow.Arrow:
            # All dates in schedule are stored as 00:00 (midnight) on the day
            # that the event is happening
            # Check if time in slot is in the same calendar day
            if timedelta(0) <= day-d < timedelta(1):
                matchedrow = e
                break

    if matchedrow is None:
        return statement(f"There is no event on {date}")

    log.debug(f"Matched row: {matchedrow}")
    eventname = matchedrow.Event
    date = matchedrow.daterange

    day = None
    if type(date) == arrow.Arrow:
        day = date.format('MMMM Do')
    elif type(date) == list:
        day = f"{date[0].format('MMMM Do')} to {date[-1].format('MMMM Do')}"

    if day is None:
        log.error("Something went horribly wrong")
        raise ValueError(f"Day should not be None here")

    return statement(f"{eventname} is happening on {day}")


@ask.session_ended
def session_ended():
    return {}, 200


# @ask.on_session_started
# def new_session():
#     log.info('Request ID: {}'.format(request.requestId))


def _get_academic_calendar():
    """
    """
    re = requests.get(ACADEMIC_CALENDAR)
    rows = html.fromstring(re.text).cssselect('table tr')

    schedule = pd.DataFrame()

    semester = None
    for row in rows:
        if 'valign' in row.attrib:
            semester = row.xpath('td')[0].text_content().strip()
        else:
            try:
                (date, name) = row.xpath('td')
                schedule = schedule.append([[date.text_content().strip(),
                                             name.text_content().strip(),
                                             semester]])
            except ValueError as e:
                print(e)

    schedule.columns = ["Date", "Event", "Semester"]
    schedule = schedule.reset_index(drop=True)

    schedule['daterange'] = schedule.apply(_get_date_from_text, axis=1)

    return schedule


MULTIDATE = re.compile(
    "(?P<month>\w+) (?P<start_day>\d{1,2})(-| )(?P<end_day>\d{1,2})")
SEMESTER_TO_YEAR = re.compile("\w+ (?P<year>\d{4})")
DATE = re.compile("(?P<month>\w+) (?P<day>\d{1,2})")


def _get_date_from_text(row):
    md = MULTIDATE.match(row['Date'])
    sty = SEMESTER_TO_YEAR.match(row['Semester'])
    if md:
        info = md.groupdict()
        year = sty.groupdict()['year']
        start_date = arrow.get(f"{year} {info['month']} {info['start_day']}",
                               "YYYY MMMM D")
        end_date = arrow.get(f"{year} {info['month']} {info['end_day']}",
                             "YYYY MMMM D")

        return arrow.Arrow.range('day', start=start_date, end=end_date)
    else:
        info = DATE.match(row['Date']).groupdict()
        year = sty.groupdict()['year']

        return arrow.get(f"{year} {info['month']} {info['day']}",
                         "YYYY MMMM D")


if __name__ == '__main__':
    app.run(debug=True)
