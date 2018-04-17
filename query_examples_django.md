FZC will not use Django, but I'm more familiar with Django at the moment, so looking at example code in Django helps me with the design phase.


## Ladders for a Game

Example use: An individual game's Ladders homepage / index / menu.

```python
def get_game_ladders(game):
    """
    With Game-Ladder links
    """
    return game.ladder_set.all()
    
def get_game_ladders_2(game):
    """
    Without Game-Ladder links
    With Ladder-ChartGroup as a one-to-many
    """
    all_groups = game.chartgroup_set
    return Ladder.objects.filter(chart_group__in=all_groups)
    
def get_game_ladders_3(game):
    """
    Without Game-Ladder links
    With Ladder-ChartGroup as a many-to-many
    """
    all_groups = game.chartgroup_set
    # Not really sure if you can do a reverse many to many
    # lookup with values() or values_list().
    list_of_ladder_lists = all_groups.values_list('ladder_list', flat=True)
    # But if so, we need to collapse into one list (sum), then
    # remove dupes (set, list).
    return list(set(sum(list_of_ladder_lists, [])))
```

Note a key difference from other rankings sites, where Ladders are an essential part of the chart hierarchy: Game -> Ladder -> Chart Sub Groups -> Charts. Here, Game-Ladder links are not strictly needed. However, such links help define an ordering of Ladders within a Game, so we have these links anyway.


## All main Ladders for all Games, sorted by Game

Example use: A site-wide Ladders menu or homepage.

```python
def get_all_main_ladders():
    """
    With Game-Ladder links.
    Each Ladder has an 'order' field specifying its ordering relative to the
    Game's other Ladders.
    """
    return Ladder.objects.filter(tier='main').order_by('game', 'order')
```


## Charts for a Game, sorted by Chart Group

Example use: Charts homepage / index / site records page / player's records page for a Game.

```python
def visit_group(group):
    if isinstance(group, LeafChartGroup):
        return dict(name=group.name, charts=group.chart_set.all())
    else:
        subhierarchy = [
            visit_group(subgroup)
            for subgroup in group.chartgroup_set.all()]
        return dict(name=group.name, subgroups=subhierarchy)

def get_game_charts(game):
    """
    Performance:
    Based on number of groups in the game
    """
    top_level_cgs = game.chartgroup_set.filter(parent=None)
    return [visit_group(group) for group in top_level_cgs]
```


## Filter Groups for a Chart, in order

Example use: Display as dropdowns on the records submission form, or on the rankings-view page for a Chart.

```python
def get_chart_filter_groups(chart):
    # This is a simplification. We'd actually want to access a custom
    # many-to-many model which includes the ordering of the
    # Filter Groups within the Chart Type.
    # https://docs.djangoproject.com/en/1.11/topics/db/models/#intermediary-manytomany
    return chart.chart_type.filter_groups.all()
```


## Filters (rules) for a Ladder, in order

Example use: Displaying the rules for the Ladder. Or figuring out which records are eligible in a certain Ladder + Chart combination.

```python
def get_ladder_rule_filters(ladder):
    # This is a simplification. We'd actually want to access a custom
    # many-to-many model which includes the ordering of the
    # Filter rules within the Ladder.
    # https://docs.djangoproject.com/en/1.11/topics/db/models/#intermediary-manytomany
    return ladder.rule_filters.all()
```


## Chosen Filters for a Record

Example use: Displaying an individual Record anywhere, such as an individual player's records page or an individual Chart's rankings page.

```python
def get_record_chosen_filters(record):
    return record.chosen_filters.all()
```


## Records for a Chart which have a particular Chosen Filter

Example use: An individual Chart's page, which has dropdowns letting you filter the records. Some filters are Chosen, some are Implied.

```python
def get_chart_records_with_chosen_filter(chart, chosen_filter):
    return chosen_filter.record_set.filter(chart=chart)
```


## Records for a Chart which satisfy a particular Implied Filter

Example use: An individual Chart's page, which has dropdowns letting you filter the records. Some filters are Chosen, some are Implied.

```python
def get_chart_records_with_implied_filter(chart, implied_filter):
    """
    With redundant Record -> Implied Filter links.
    """
    return implied_filter.record_set.filter(chart=chart)
    
def get_chart_records_with_implied_filter_2(chart, implied_filter):
    """
    Without redundant Record -> Implied Filter links
    
    Performance:
    '|' (or) ops: One per Chosen Filter which can imply this Implied Filter.
    In the case of GX machine filters, this can be a lot. Non Custom is
    implied by 41 machines. Custom is implied by 15000+ machines.
    """
    cfs = implied_filter.chosen_filters.all()
    records = Record.objects.none()
    for cf in cfs:
        records_with_cf = cf.records.filter(chart=chart)
        records = records | records_with_cf
    return records
```

Another approach to see if ONE Record qualifies for an Implied Filter: Take the Filter Group of any Chosen Filter that points to the Implied Filter. Then take the Record's Chosen Filter for that Filter Group. See if that Chosen Filter is in the IF's Chosen Filters. The problem is we seem to require iteration over Records, which is generally a lot worse than iteration over Filters (well, except in the GX machines case).

If redundant Record -> Implied Filter links are made, these links must be updated when a record is submitted, when a record's Chosen Filters get updated, and when a new Implied Filter is created. Update performance depends on the number of Implied Filters, including custom ones made by users (if we allow those).

If user-made Implied Filters are allowed, then a possible performance balance would be to only make redundant Record -> Implied Filter links for non-user-made IFs, and not for user-made IFs (which perhaps should get their own subclass table). The idea here is that writing the redundant links for all users' IFs is a performance pain because there can be say, 100 users * 10 IFs. On the other hand, on any given viewing of a rankings page, the user only has a few of their own IFs active at most. Say they have 5 IFs at once for that chart, which is already a lot. 5 slow-ish joins for those 5 IFs is quite reasonable.

A general design note: These Filters are very general compared to say, having filter1 / filter2 / filter 3 / etc. DB columns in the Records table. The generality means Filters end up using a many-to-many relation instead of some simple columns. Advantage: we can define as many kinds of Filters as we want. Disadvantage: it's more difficult to efficiently do Filter operations.


## Records in a Chart for a Ladder

Example use: A Chart's rankings view under a particular Ladder. The Ladder may have multiple Chosen and Implied Filters.

```python
def get_chart_records_with_filter(chart, filter):
    if isinstance(filter, ChosenFilter):
        return get_chart_records_with_chosen_filter(chart, filter)
    else:
        return get_chart_records_with_implied_filter(chart, filter)

def get_chart_records_for_ladder(chart, ladder):
    records = Record.objects.filter(chart=chart)
    for filter in ladder.rule_filters.all():
        records = records & get_chart_records_with_filter(chart, filter)
```


## All Records in a Chart from a User

Example use: A User's record history for a particular Chart.

```python
def get_chart_records_for_user(chart, user):
    return Records.objects.filter(chart=chart, player=user)
```
